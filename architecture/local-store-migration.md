# Migrating a Local Store Without Losing Data

_Moving a client app's saved state from one store to another (a JSON file and preference keys into a database, one schema version to the next) so that a partial failure, a crash, a second running copy or an unreadable file never silently throws the user's data away._

A server migration runs once, watched, with a backup. A client migration runs on every user's
machine, unwatched, at launch, against files in whatever state that machine left them. It will
meet every partial failure you did not plan for. The rules below make each run safe to repeat,
so a failure costs a retry instead of the user's data.

---

## The trap

The obvious design gates the whole import on "the destination is new" and fills in defaults
for anything missing:

```text
open destination            -> isNew = the file did not exist
if isNew: import everything from the old store, then delete the old store
load roster                 -> empty? seed a default item and save
```

Every step of it is fine until one fails halfway:

| What happens | Result with the obvious design |
| --- | --- |
| The roster saves, then renaming the old file fails | The import throws before the settings run. The next launch sees a destination that is not new and never retries, so the settings are lost. |
| The roster save throws (a hand-edited file with duplicate ids) | The launch continues, the roster is empty and a default item is seeded and saved. The destination now has a roster, so the old one is never imported. |
| A settings write fails silently, then the old key is deleted | The setting is gone from both stores. |
| Two copies of the app start together on first launch (a dev build and an installed one share the container) | The second copy runs the schema step the first one just applied, fails with "table already exists", treats the good file as corrupt and replaces it. |
| Any open error (busy, permissions, I/O) is treated as "file is corrupt" | A good database is moved aside and the app starts empty. |

Each outcome looks the same to the user: their data is gone, and nothing says why.

## The rules

1. **Gate each item on "not yet in the destination", never on "the destination is new".**
   Import the roster only while the destination has no roster. Import each setting only while
   the destination lacks that key and the source has it. The import then becomes idempotent:
   running it on every launch is harmless, and a partial failure is retried automatically.
2. **Remove the source only after the destination write succeeded.** A write that swallows its
   own errors must be read back before the source key is deleted. Rename old files
   (`<name>.migrated`) instead of deleting them. That gives a manual way back.
3. **Never seed defaults over a pending source.** While the destination is empty and the old
   source still exists, report "unavailable" instead of "empty". A thin repository guard in
   infrastructure does this without touching the use case that seeds defaults. Seeding is a
   write, and that write is what makes the idempotence gate close forever.
4. **Keep items independent.** A roster failure must not skip the settings. Run every item,
   collect the first error and rethrow it at the end.
5. **A failed import does not stop the launch.** Log it and retry on the next launch; rules 1–4
   make that retry safe. Opening the destination may still be fatal: a store that cannot be
   created is a broken machine, not a migration problem.
6. **Move a store aside only when the file itself is unusable.** Only three cases count: corrupt,
   not a database, and a schema version newer than this build knows. Every other error (busy,
   permission denied, cannot open, I/O) stops the app and leaves the data where it is. Never
   overwrite: rename with a timestamp (`<store>.corrupt-<unix-seconds>`), including sidecar
   files such as `-wal` and `-shm`. Log every move-aside, or "my data vanished" cannot be
   diagnosed.
7. **Read the schema version inside the write transaction that applies the step.** Take the
   write lock first (`BEGIN IMMEDIATE` in SQLite), then read the version, then apply exactly one
   step and bump it. A version read before the lock is stale by the time the step runs. Append
   new steps; never edit a shipped one.

## Checklist

- [ ] The import runs on every launch and is a no-op once done.
- [ ] Each item has its own "already in the destination?" gate.
- [ ] No source is deleted or renamed before its destination write is confirmed.
- [ ] Default seeding cannot run while a legacy source exists.
- [ ] One item's failure does not skip the others; the error still surfaces.
- [ ] A failed import is logged and retried. The app still starts.
- [ ] Move-aside happens only for corrupt, not-a-database and future-version files. It is logged and never overwrites.
- [ ] Schema steps re-read the version under the write lock.
- [ ] Tests cover each partial failure: a stale `.migrated` file already present, the roster save throwing, the destination already holding a value, a second run, and two connections migrating the same file.

## Measured example — MyPets, JSON and preference keys into SQLite

A macOS menu-bar app moved its roster (`pets.json`) and two UserDefaults keys into one SQLite
file (`mypets.sqlite`). The plan used the obvious design: import only into a fresh database.
Reviews of the implementation found five data-loss paths before merge. Every path is in the
table above, and none was covered by the plan's tests. The shipped importer follows rules 1–4.
A `PendingLegacyRosterGuard` repository reports the roster unavailable while `pets.json`
still exists (rule 3). The composition root logs importer failures under a `storage` category
and continues (rule 5). `MyPetsDatabase.open` moves a file aside only for `SQLITE_NOTADB`,
`SQLITE_CORRUPT` or an unsupported version (rule 6), and each schema step re-reads
`PRAGMA user_version` inside `BEGIN IMMEDIATE` (rule 7). The whole change added 22 commits and
173 tests.

## Related

- [client-state-persistence.md](./client-state-persistence.md) — who owns each persisted value in a multi-runtime client; decide that before designing the migration.
- [repository-pattern.md](./repository-pattern.md) — the port the guard in rule 3 wraps.
- [../desktop/swift-modular-architecture.md](../desktop/swift-modular-architecture.md) — the system SQLite wrapper in Swift that these rules ran on.
