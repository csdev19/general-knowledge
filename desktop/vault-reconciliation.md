# Vault reconciliation

*Bringing a filesystem-first library back in line with its folder: follow names only, never
infer identity, never delete, and list what is ambiguous. On demand, idempotent, and previewed
as a before → after table.*

A [library vault](./library-vault.md) keeps the media file as the source of truth and its
metadata in a sidecar keyed by the file name. The two drift apart, and nothing realigns them
by itself:

- the app changes its naming rule (for example, the file starts following the title), and
  every item created under the old rule now disagrees;
- the user renames, moves, copies or deletes files in the file manager;
- a crash leaves a half-done operation behind (a staging part, a record without media);
- an older build wrote names or titles the current one no longer produces.

Keeping the registry inside the folder avoids a global index that points at the wrong place.
It does not remove the drift: an external rename still separates a file from its sidecar.

## The rule: follow names only

The reconciler pairs a file with a record **by name and nothing else**. It never guesses that
`Demo final.mp4` "is" the record `Demo.json` because the duration, size or creation date
match.

Identity inference looks cheap and turns out to be the expensive part. Each heuristic
(same duration, same size, same date, same hash prefix) has a fixture that pairs two different
items, and every review round finds a new one. A false pairing moves one item's tags, edits and
history onto another item, silently. A missed pairing only leaves a record unattached, and
that is recoverable.

What follows from the rule:

| Situation | What the reconciler does |
| --- | --- |
| Media with no record | Adopts it as a new item under its disk name |
| A record whose media is gone | Moves the record and its companions to a quarantine folder, with a manifest; never deletes |
| A record whose title and file name disagree | Renames the file to follow the title (the app wins for an item it manages) |
| An old automatic title | Replaces it with the current default name, then renames |
| A file renamed in the file manager | Becomes a new item; the old record goes to quarantine. Manual reassignment is a separate feature |
| Two items claiming one identity, two files on one name, unreadable records | Listed for the user, never touched |

"Never delete" is what makes the rest affordable. The reconciler only has to be right
about what it changes. What it is unsure of goes to the list or the quarantine folder, where
the user can still act on it.

## Shape of the operation

- **On demand, not on launch.** A banner ("N files don't match") and a button in settings start
  it. A silent fix on startup would rename files under the user's feet. A banner explains a problem the
  user already has.
- **Scan, then fix.** The scan only reads and returns findings; the fix runs one pass. Running
  the scan again after a fix must find nothing fixable: that is the idempotence test, and it
  covers future drift too.
- **Strict reads.** The scan reads sidecars with an explicit outcome: ok, missing, unreadable,
  invalid JSON or invalid schema. A corrupt sidecar is listed and **never rewritten** by anyone,
  including the normal list, backfill and editor paths. Treating "unparseable" as "missing"
  mints a new identity over the old record and loses it.
- **Maintenance mode refuses work instead of queueing it.** While the fix runs, other vault
  writes are refused with a clear message, and editors retry when it ends. The fix does not
  start during a recording or export. The admission check and the in-flight counter change in one
  synchronous step, so nothing slips in between.
- **Write the manifest before moving to quarantine.** If the fix stops halfway, the next run
  first replays the pending manifests, and does nothing else when one cannot be read.
- **Plan renames as a graph.** Renames form chains (`A → B` while `B → C`) and cycles
  (`A ↔ B`). Order the chains from the free end, and break a cycle with a ` (2)` suffix instead of a
  temporary name that a crash could strand.
- **Place files without replacing.** Every move of media into the vault goes through one
  primitive that refuses to overwrite: [no-replace file placement](./no-replace-file-placement.md).

## Preview what it will do, on real data

Unit fixtures are built from the inputs the author expected, so they cannot fail on the ones
the author did not imagine. Before handing a reconciler or migration over, run its scan on a
**copy** of a real vault and read a sample of the proposed output (the new names, not just
the counts).

### Measured example

On a 561-file vault, the scan reported 527 old automatic titles, and every unit test passed.
The owner's first look at the proposed names found rows like
`Recording — 9 14 2026, 10 57 31 AM (edited).mp4`. Exports carried an ` (edited)` suffix after the
old title, so the detector did not recognize them. They kept the old title, and the file-name
sanitizer turned `/` and `:` into spaces. The fix names an export after its source's new name
plus the suffix. Reading twenty proposed names would have caught it before the handoff.

## Show the preview as a table

A list of sentences like `old-name.mp4 → New name.mp4` does not scale: past a dozen rows the
reader cannot tell which side is which. Use a two-column table, **Now** and **After the fix**,
with:

- the arrow only on rows that rename. Other rows say in muted text what happens ("added to the
  library", "record moves to quarantine");
- the extension dropped when it is the same on both sides, with a small kind icon instead;
- tabular figures, so dates in names line up row under row and the pattern reads at a glance;
- a second table, **File** and **Why it stays**, for what is listed and not fixed.

## Accepted limits

- A rename in the file manager makes a new item; its metadata waits in quarantine.
- Swapping two names in the file manager swaps their metadata, because the names are the
  identity.
- A user title that happens to look automatic is replaced as automatic.

## Related

- [library-vault.md](./library-vault.md): the storage model being reconciled
- [no-replace-file-placement.md](./no-replace-file-placement.md): the placement primitive
  every fix goes through
- [product-philosophy.md](./product-philosophy.md): the vault model, one folder per library
- [../conventions/verifiable-handoffs.md](../conventions/verifiable-handoffs.md): why a check
  that could not have failed proves nothing
