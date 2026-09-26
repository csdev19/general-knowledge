# Tauri: where business logic and data live

_How to split a Tauri 2 app between the webview and the Rust core — four placements, what
each implies, which one shipping apps use, and a default for monorepos whose domain is
already TypeScript._

> Companion to [electron-vs-tauri.md](./electron-vs-tauri.md), which decides _whether_ to use
> Tauri. This doc assumes you already chose it.

## The constraint that shapes everything

In Electron the main process is Node, so the same TypeScript runs on both sides of IPC. In Tauri
the privileged side is **Rust**. TypeScript use cases cannot run there, so "where does the domain
run?" becomes a real decision instead of a default.

Tauri's security model settles half of it: code in the webview "has only access to exposed system
resources via the well-defined IPC layer", while Rust has full system access, and data crossing
that boundary should be strongly defined. ([Tauri security](https://v2.tauri.app/security/))
**Storage and privileged operations belong in Rust**, behind narrow commands. The open question is
only where the _business rules_ run.

## Four placements

| | A. SQL from the webview | B. Everything in Rust | C. JS sidecar | D. Hybrid |
| --- | --- | --- | --- | --- |
| Use cases run in | webview (TS) | Rust | Node/Bun process | webview (TS) |
| Storage owned by | Rust plugin, but the webview writes the SQL | Rust | sidecar | Rust, fixed statements |
| Reuses a TS domain | ✅ | ❌ rewritten in Rust | ✅ | ✅ |
| Trust boundary | ⚠️ any SQL crosses it | ✅ strongest | ⚠️ your own IPC to secure | ✅ narrow typed commands |
| Transactions | ❌ (see below) | ✅ | ✅ | ✅ |
| Bundle / memory | ✅ | ✅ | ❌ ships a JS runtime | ✅ |

### A — `@tauri-apps/plugin-sql` from the webview

The official plugin exposes `select`/`execute` to JavaScript over sqlx. It has migrations and
capability-scoped permissions, but `execute` takes the **SQL string from the webview**, and it
has **no transaction API**: statements go through a pool, so a `BEGIN` and the statement after it
may reach different connections
([plugins-workspace#886](https://github.com/tauri-apps/plugins-workspace/issues/886)).
Community forks (`tauri-plugin-rusqlite2`, `tauri-plugin-sqlite`) add transactions, but the SQL
still originates in the least-trusted process. Fine for a prototype; a poor pattern to template.

### B — The core in Rust

Domain, use cases and storage in Rust crates; the webview only renders and calls commands. This
is what the reference Tauri apps do:

- **GitButler** — all logic in `crates/`; the same core powers the desktop app and the `but`
  CLI. ([repo](https://github.com/gitbutlerapp/gitbutler))
- **Yaak** — logic in Rust crates reused by `crates-tauri`, `crates-cli` and `crates-server`.
  ([repo](https://github.com/mountain-loop/yaak))
- **Spacedrive**, **Cap** — Rust core, TypeScript types generated from Rust.

The common thread: the Rust core is **reused outside the webview** (a CLI, a server, mobile), or
the work is heavy (git, file indexing, media). If neither holds and the domain already exists in
TypeScript, B means maintaining the same rules in two languages.

### C — A Node/Bun sidecar

Tauri documents it ([Node.js as a sidecar](https://v2.tauri.app/learn/sidecar-nodejs/)), with
caveats: you design and secure the IPC (stdio, sockets or localhost — Tauri discourages
localhost), manage another process's lifecycle, and ship a JavaScript runtime. It gives up most
of what Tauri is chosen for. Reserve it for a hard dependency on a Node-only library.

### D — Hybrid: TS use cases, Rust-owned storage

The use cases run in the webview, unchanged. The repository **port** is implemented in
TypeScript by an adapter whose every method is one typed Rust command; Rust holds the connection
and the fixed, parameterized statements.

```
webview (TS)                                       Rust core
use case ──► TauriRepository ──invoke(typed)──►   command ──► SQL (fixed) ──► SQLite
             implements the port                   validates types, owns transactions
```

It mirrors Electron's shape — an adapter behind IPC — with the cut in a different place: in
Electron the use cases sit in the main process and the renderer calls them; here the use cases
sit in the renderer and the repository sits behind the boundary.

**The honest cost.** Domain rules (schemas, invariants) run in the webview. Rust guarantees types
(serde) and SQL constraints, and can re-check the few rules that matter for integrity (e.g. a
title length). A compromised webview could bypass the rest, but only through the narrow commands
and only over the local user's own data. For a single-user local-first app that is usually
acceptable; for multi-tenant or security-sensitive logic, move that logic to Rust (toward B).

**The migration path.** D does not paint you into a corner: move a use case to Rust one command at
a time when it gets heavy or needs to be shared with a non-TS consumer. The UI does not change.

## Default

- The domain is already TypeScript and shared with a server or web app → **D**.
- The core must be reused by a CLI/server in Rust, or the work is compute-heavy → **B**.
- Prototype, single developer, no template → **A** is acceptable.
- A Node-only dependency you cannot replace → **C**, reluctantly.

## Making D work well

- **Typed commands.** Generate the TypeScript bindings from the Rust signatures
  ([tauri-specta](https://github.com/specta-rs/tauri-specta)), commit the output, and fail CI when
  regenerating changes it. tauri-specta for Tauri 2 is still a release candidate: pin `specta`,
  `specta-typescript` and `tauri-specta` with `=`, because RCs break between versions.
- **Numbers.** Specta refuses to export `i64`/`u64` (they need `bigint`). Epoch-millisecond
  timestamps fit exactly in `f64`; specta types an `f64` as `number | null` (serde maps NaN to
  null), so convert once in the adapter and treat `null` as a broken contract.
- **Tri-state patches.** "Absent" vs `null` does not survive a typed JSON binding; spell it out
  (`{ kind: "keep" } | { kind: "set", value: string | null }`).
- **Transactions in Rust.** A read-modify-write update is one command running in one transaction —
  never two round-trips from the webview.
- **Migrations.** An ordered list of SQL steps gated on `PRAGMA user_version`, each in its own
  transaction. Never edit a shipped step.
- **Test at the wire.** Rust unit tests cover the SQL against an in-memory database. On the TS
  side, `mockIPC` from `@tauri-apps/api/mocks` fakes the core at the IPC layer, so the tests also
  check command names and argument keys — then run the real use cases on the adapter.

## Native chrome still needs Rust

Whatever the placement, anything the OS draws — the tray menu, native menus, notifications — is
built in Rust. To translate it without a second copy of the strings, embed the web app's message
catalogs at compile time (`include_str!` of the JSON) and look keys up in Rust.

## Reopen when

- A second, non-TypeScript consumer of the domain appears (Rust CLI, native mobile) → move the
  core to Rust (B).
- Integrity rules start mattering against a hostile webview (multi-user data, licensing) → move
  those rules into the commands.
- tauri-specta ships a stable 2.x → drop the `=` pins.
