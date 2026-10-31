# No-replace file placement

*Moving a file into a user-visible folder without ever overwriting what is already there, in
Node.js, where `rename` replaces silently and there is no rename-without-replace.*

A local-first app writes into a folder that it does not control: the file manager, a sync
client or another process can create a file at any path at any moment. `fs.rename(from, to)`
replaces `to` without asking, and a "check, then rename" pair leaves a window where an arriving
file is destroyed. Node exposes neither `renameat2(RENAME_NOREPLACE)` nor macOS
`renamex_np(RENAME_EXCL)`, nor an unlink by file handle.

The fix is one primitive that every placement goes through: finalizing a recording, renaming
an item, saving a screenshot or GIF, moving a record to quarantine. No call site gets its own
version.

## The primitive

1. **Duplicate without replacing.** `link(from, to)` creates `to` only if it does not exist
   (EEXIST otherwise), atomically. When the volume has no hard links (EPERM, ENOTSUP,
   EOPNOTSUPP, ENOSYS, EXDEV, EMLINK: exFAT, some network shares, across volumes), fall back
   to an **exclusive copy**: `open(to, "wx")`, copy in chunks, and honour `bytesWritten`,
   because a short write is legal. Then flush the file.
2. **Check the destination still holds what you made.** Read its identity (device and inode)
   and compare it with the one captured from your own handle or link. If it differs, a stranger
   replaced it. That file is theirs, and the source keeps the bytes.
3. **Remove the source only by identity.** Unlink a name only while it still holds the file whose
   identity this operation captured. Never unlink on an identity merely observed at a path,
   and never when the path cannot be read.
4. **Flush the directory** after the link and after the unlink, so a power loss does not leave
   both names or neither.

The result is `placed` or `taken`. Taken is a normal outcome: the caller keeps the old name and
tells the user. It is not an error.

## The details that bite

- **Read identities as bigints.** `stat({ bigint: true })`: Windows file ids overflow a
  double, so two different files can compare equal as numbers.
- **A source that cannot be removed.** The bytes are at the destination, but the source unlink
  failed. For a staging file, rename it aside (`.placed`), or write a `<source>.placed.json`
  marker so the crash-recovery sweep skips it. Otherwise it is recovered a second time. For a
  vault file being renamed, roll the placement back instead. The marker never authorizes
  deleting anything: a size check that "confirms" the destination is a guess.
- **Case-only renames on case-insensitive volumes.** `Demo.mp4 → demo.mp4`: the destination
  "exists" because it is the same file. Go through a unique temporary sibling, two no-replace
  steps, and move back if the second is taken.
- **Cleanup only touches what the operation created.** A failed attempt removes its own
  temporary file by identity, never the destination it was aiming at.
- **The one replacing placement is explicit.** When the user answers "Replace" in a save dialog,
  duplicate into an exclusively created temporary sibling, then `rename` it over the target.
  Replacing a file with itself is a no-op, or the bytes end up under the hidden temp name.

## Threat model, stated

It is guaranteed against the app's own concurrency, a crash at any point, an I/O error at any
call, and a file present at the destination or arriving before the no-replace step.

It is **not** guaranteed against an external process that swaps a path in the instant
between the identity check and the `unlink` that follows it. Closing that window needs a
syscall Node does not expose. Write that boundary down in the decision record, so reviewers
judge the design against it and do not reopen it every round.

## Related

- [vault-reconciliation.md](./vault-reconciliation.md): the bulk operation that relies on it
- [library-vault.md](./library-vault.md): the folder it writes into
- [media-pipeline.md](./media-pipeline.md): staging files and crash recovery
