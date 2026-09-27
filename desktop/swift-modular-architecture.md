# Native macOS in Swift — DDD layering with SwiftPM modules

_How the domain → application → infrastructure layering of [architecture/](../architecture/)
maps onto a native SwiftUI app, which parts of hexagonal pay for themselves in a
single-process desktop app, and the Swift-specific details that make it hold._

The TypeScript docs in `architecture/` keep layers apart with packages and convention. Swift
has a stronger tool: **a SwiftPM target is a visibility boundary the compiler enforces.** Most
of this page follows from that one fact.

---

## Shape

```text
App/                      Xcode app target: composition root + shell (menu bar, windows)
Modules/                  local SwiftPM package, all product code
  Package.swift             the dependency rules, as code
  Sources/
    SharedKernel/           typed IDs, time port — nothing feature-specific
    DesignSystem/           semantic tokens
    <Feature>Domain/        entities, value objects, errors, driven ports
    <Feature>Application/   one struct per use case
    <Feature>Infrastructure/ adapters implementing the ports
    <Feature>UI/            @Observable model + views
  Tests/                    one test target per layer
project.yml               XcodeGen spec; the .xcodeproj is generated, never committed
```

The arrows are declared in `Package.swift`:

```swift
.target(name: "<Feature>Domain", dependencies: ["SharedKernel"]),
.target(name: "<Feature>Application", dependencies: ["<Feature>Domain", "SharedKernel"]),
.target(name: "<Feature>Infrastructure", dependencies: ["<Feature>Domain"]),
.target(name: "<Feature>UI", dependencies: ["<Feature>Application", "<Feature>Domain", "DesignSystem"]),
```

A target can import only what it lists, and a symbol not marked `public` is invisible outside
its target. So `Domain` importing an adapter, or one feature reaching into another's internals,
is a **compile error** — no lint rule, no review comment.

## Per-feature targets, not per-layer

The TypeScript template is layer-first: one `domain` package holding every feature's domain.
In Swift, prefer **four targets per feature** (`TodosDomain`, `TodosApplication`, …). The target
is also the visibility boundary, so per-feature targets keep each bounded context's internals
private from the others for free. Layer-first targets would put every feature's `internal`
types in one namespace.

Cost: a new feature is four targets (plus tests) in `Package.swift` and four products in
`project.yml`. That is copy-paste of one block.

Folders inside the app target are the option to avoid: nothing stops a `Domain/` folder from
importing SwiftUI, so the rules become convention again.

## How much hexagonal

Keep **driven ports**: what the core needs from outside — storage, time, notifications, a
server — is a protocol in the domain (or `SharedKernel` when shared), implemented in
infrastructure and chosen in the composition root. This is what buys tests without disk and
swapping JSON for SwiftData without touching the domain.

Drop, until a trigger appears:

- **Driving ports** (an input protocol per use case). With SwiftUI as the only caller, each
  one is a single-implementation abstraction. Use cases are concrete structs the UI calls.
- **DTOs and mappers between application and UI.** Everything is in one process; the domain's
  immutable value types can cross into views.
- **Command/query buses.** Nothing to route.

**Trigger to add driving ports:** a second entry point into the same feature — App Intents /
Shortcuts, a widget, a CLI, an XPC service. Then both adapters call through one protocol.

## One exception to "no mappers": the persistence boundary

Do **not** make domain entities `Codable` and write them to disk. The file on a user's machine
outlives every build of the app; renaming a property of the entity would make saved data
unreadable after an update. Map to an explicit, versioned record in infrastructure:

```swift
struct TodoFile: Codable {          // infrastructure only
    var version = 1                 // bump + migrate on shape changes
    var todos: [TodoRecord]
}
```

This is the one mapper worth its lines in a desktop app, for the same reason a server keeps DB
rows apart from domain types.

## Swift details that make the layers hold

- **Value objects with typed throws.** `init(_ raw: String) throws(TodosError)` means a
  `TodoTitle` that exists is valid; no layer re-validates it, and callers get the concrete
  error type without casting.
- **Phantom-typed IDs.** `struct EntityID<Entity>: Hashable, Codable { let rawValue: UUID }` —
  `EntityID<Todo>` and `EntityID<Note>` are different types. `Entity` is never stored, so the
  ID stays `Sendable` whatever the entity is.
- **Entities as structs with `private(set)` state** and intention-revealing mutating methods
  (`complete(at:)`, `reopen()`). Value semantics make "the UI edited my entity" impossible.
- **Time is a port.** Call it `DateProvider`, not `Clock` — `Clock` is a standard library
  protocol and the name collision is confusing.
- **Adapters as actors.** A file-backed repository as an `actor` serializes access with no
  locks; the port is `async` anyway because real adapters are.
- **Default isolation per target.** Swift 6.2 lets a target default to the main actor:
  `swiftSettings: [.defaultIsolation(MainActor.self)]`. Use it for UI and design-system
  targets; leave domain, application and infrastructure nonisolated so their types stay
  `Sendable` values usable from any context.
- **Ubiquitous language in a file.** A root `CONTEXT.md` with one table per bounded context,
  including words to avoid. In Swift, list *task* as avoided when the concept is a to-do:
  it collides with `Task`.

## Testing

`swift test --package-path Modules` runs domain, use-case and adapter tests in seconds, with no
Xcode project, app host or simulator — the app target holds nothing worth unit-testing. Use
cases get an in-memory `actor` implementing the port; the time port gets a hand-moved fake.
In CI this keeps the macOS runner minutes (the expensive ones) short.

## Project generation

Commit `project.yml` (XcodeGen) and ignore `*.xcodeproj`. The `.pbxproj` is the part of an Xcode
project that fights renames and merges; generating it makes a template rename a text
replacement and removes project-file conflicts. Local packages (`Modules/`) need no regeneration
when files are added; the app target does.

## Menu bar app notes

- **`MenuBarExtra(...).menuBarExtraStyle(.window)`** is the overlay: a popover-like window under
  the status item. Put Settings in it as a page, not a separate window — in an agent app
  (`LSUIElement`, no Dock icon) a Settings window tends to open behind everything.
- **Load data from the label, not only the overlay.** The status item exists from launch; the
  overlay's views only on first click. A count shown in the menu bar needs a `.task` on the
  label.
- **Gate Sparkle on configuration.** Start the updater only when the build carries both
  `SUFeedURL` and `SUPublicEDKey`; otherwise a fresh clone shows "the updater failed to start"
  on every launch. Feed both from build settings (`$(APP_FEED_URL)`) so CI injects them.
- **UI automation cannot always reach the status item.** With a crowded menu bar the item sits
  behind the notch; `AXPress` reports success and nothing opens. Verify the overlay by hand, or
  test its model instead.

Shipping the app — signing, notarization, DMG, Sparkle feed — is
[distribution/swift-xcode-macos.md](../distribution/swift-xcode-macos.md).

## Related

- [swift-macos-app-lifecycle.md](./swift-macos-app-lifecycle.md) — the composition root builds, the app delegate acts: AppKit side effects taken in the `App` initializer silently stop mouse delivery.
