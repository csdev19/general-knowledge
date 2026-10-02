# From React/TypeScript to native Swift — equivalences and patterns

_What each piece of a TanStack/Zod/Zustand stack becomes in SwiftUI, and the rule that keeps a
React developer from rebuilding the JavaScript stack library by library. Companion to
[swift-modular-architecture.md](./swift-modular-architecture.md), which covers the package
layout; this page covers the code inside the packages._

Swift needs fewer libraries than React. Reactivity, state and forms come with the language and
with SwiftUI. The table is the quick map; each section below is the pattern.

| JS/TS                                | Swift equivalent                                                  | Kind                            |
| ------------------------------------ | ----------------------------------------------------------------- | ------------------------------- |
| Zod                                  | `Codable` + types validated in their `init`                       | Native                          |
| Zustand                              | `@Observable` + `Environment`                                     | Native (macOS 14+ / iOS 17+)    |
| Zustand `persist`                    | `@AppStorage`, or `swift-sharing` (`@Shared(.appStorage)`)        | Native / library                |
| Redux, stores with effects           | The Composable Architecture (TCA)                                 | Library (Point-Free)            |
| TanStack Router                      | `NavigationStack` / `NavigationSplitView` + `Scene`s              | Native                          |
| TanStack Query                       | `async/await` + `.task { }` + a cache in the store                | Native; no mature equivalent    |
| oRPC client / server functions       | `swift-openapi-generator` from the OpenAPI spec oRPC exports      | Library (Apple)                 |
| react-hook-form                      | Bindings (`@Bindable`, `$model.field`) + `Form`                   | Native                          |
| children / slots                     | `@ViewBuilder` parameters                                         | Native                          |
| HOCs / wrappers                      | `ViewModifier`                                                    | Native                          |
| React Context                        | `@Environment` + `@Entry`                                         | Native                          |
| Headless components / variants       | `ButtonStyle`, `LabelStyle`, `ToggleStyle`                        | Native                          |
| Interfaces / ports                   | `protocol`                                                        | Native                          |
| DI container                         | `Environment`, or `swift-dependencies`                            | Native / library                |
| Turborepo monorepo                   | Swift Package Manager with local packages                         | Native                          |
| Drizzle + SQLite                     | SwiftData, GRDB or SQLiteData                                     | Native / library                |
| Vitest                               | Swift Testing (`@Test`, `#expect`)                                | Native                          |

**Starting rule:** begin with `@Observable` + `Environment` + local SwiftPM packages, and add
TCA, `swift-dependencies` or `swift-sharing` only when the concrete problem they solve shows up.

## Validation: Zod → Codable + "parse, don't validate"

There is no dominant Zod in Swift and none is needed. `JSONDecoder` already throws when the JSON
does not match the type, so parsing is free. Business rules (format, ranges, lengths) go in the
type's `init`.

```swift
enum ValidationError: Error {
    case invalidEmail
    case tooShort(min: Int)
}

struct Email: Codable, Hashable {
    let value: String

    init(_ raw: String) throws {
        let trimmed = raw.trimmingCharacters(in: .whitespaces)
        guard trimmed.contains("@") else { throw ValidationError.invalidEmail }
        value = trimmed
    }

    // Validates when decoding JSON too.
    init(from decoder: Decoder) throws {
        try self.init(try decoder.singleValueContainer().decode(String.self))
    }

    func encode(to encoder: Encoder) throws {
        var c = encoder.singleValueContainer()
        try c.encode(value)
    }
}

struct User: Codable {
    let id: UUID
    let email: Email   // if it exists, it is valid
}
```

Point equivalences with Zod:

- `z.object({...})` → a `struct` conforming to `Codable`.
- `z.infer<typeof schema>` → does not exist: the type _is_ the schema.
- `.optional()` → a `T?` property; `.default(x)` → `decodeIfPresent(...) ?? x` in a custom
  `init(from:)`.
- `z.enum([...])` → `enum Status: String, Codable`.
- `z.union` / `discriminatedUnion` → an `enum` with associated values, decoded by a `type` field.
- `.transform()` → a computed property, or an `init` that converts.
- `safeParse` → `try?`, or a `Result<T, Error>`.

Recommended: small validated types (`Email`, `NonEmptyString`, `Money`, `KeyCombo`) in the
`Domain` package. No other code validates them again.

## State: Zustand → @Observable

`@Observable` (the Observation framework, macOS 14+ / iOS 17+) is a reactive store with
fine-grained tracking: a view re-renders only if it read the property that changed, like a
Zustand selector.

```swift
@Observable @MainActor
final class TaskStore {
    var tasks: [TaskItem] = []
    var filter: Filter = .all

    var visible: [TaskItem] {          // a derived selector
        tasks.filter(filter.matches)
    }

    func add(_ t: TaskItem) { tasks.append(t) }
    func toggle(_ id: TaskItem.ID) {
        guard let i = tasks.firstIndex(where: { $0.id == id }) else { return }
        tasks[i].done.toggle()
    }
}

@main struct MyApp: App {
    @State private var store = TaskStore()   // owns the store
    var body: some Scene {
        WindowGroup { ContentView() }
            .environment(store)               // the Provider
    }
}

struct ContentView: View {
    @Environment(TaskStore.self) private var store  // useStore()
    var body: some View {
        List(store.visible) { Text($0.title) }
    }
}
```

When to scale up:

| Need                                                 | Tool                                 |
| ---------------------------------------------------- | ------------------------------------ |
| State of one view                                    | `@State`                             |
| A shared store                                       | `@Observable` + `.environment`       |
| Persisting simple preferences                        | `@AppStorage`                        |
| Persisted global state shared across features        | `swift-sharing` (`@Shared`)          |
| Redux-style flow, effects, exhaustive tests          | The Composable Architecture (TCA)    |

TCA is powerful but opinionated, with a learning curve. For menu bar utilities and mid-sized
apps, `@Observable` is usually enough.

A persistence detail worth copying: one `@Observable` settings object whose properties persist in
`didSet`, injected through `Environment`, instead of `@AppStorage` scattered through views. A
service that must react the moment a preference changes (a running session that re-evaluates a
battery rule, a hot key that re-registers on a new combination) can then observe the store or be
handed a callback, which `@AppStorage` in a view cannot offer.

## TanStack Start → routing, data and contracts, separately

TanStack Start is a full-stack web framework; in a native app it splits into three pieces.

### Routing

Value-based navigation with typed routes, close to TanStack Router's:

```swift
enum Route: Hashable {
    case project(Project.ID)
    case settings
}

struct RootView: View {
    @State private var path: [Route] = []
    var body: some View {
        NavigationStack(path: $path) {
            HomeView()
                .navigationDestination(for: Route.self) { route in
                    switch route {
                    case .project(let id): ProjectView(id: id)
                    case .settings: SettingsView()
                    }
                }
        }
    }
}
```

On macOS, think in `Scene`s as well as routes: `WindowGroup` (windows), `Window` (a single
window), `Settings` (preferences, ⌘,) and `MenuBarExtra` (menu bar apps). Sidebar + detail is
`NavigationSplitView`.

### Data fetching (loaders / TanStack Query)

`.task { }` runs when the view appears and cancels itself when it disappears, like a loader with
an `AbortController`:

```swift
struct ProjectView: View {
    let id: Project.ID
    @State private var project: Project?
    @State private var error: Error?

    var body: some View {
        content
            .task(id: id) {                    // re-runs when id changes
                do { project = try await api.project(id) }
                catch { self.error = error }
            }
    }
}
```

There is no mature TanStack Query for Swift. The usual pattern is an `@Observable` store that
caches by id and exposes `isLoading` / `error`.

### Contracts with the backend (oRPC)

Keep the contract-first design: the contract lives in TypeScript and the Swift client is generated.

1. Export the OpenAPI spec from oRPC.
2. Add `swift-openapi-generator` (Apple) to the `Infrastructure` package with the spec and an
   `openapi-generator-config.yaml`.
3. The build generates a typed `Client` with `Codable` models.
4. Wrap that client behind a `Domain` protocol (a port), so views never depend on generated code.

## Reactive forms → native bindings

SwiftUI is two-way binding by design, so there is no react-hook-form to replace. The form model
is an `@Observable`, the view receives it with `@Bindable`, and `$form.field` is the binding.

```swift
@Observable
final class SignupForm {
    var email = ""
    var password = ""
    var touched: Set<Field> = []

    enum Field { case email, password }

    var emailError: String? {
        (try? Email(email)) == nil ? "Invalid email" : nil
    }
    var passwordError: String? {
        password.count < 8 ? "At least 8 characters" : nil
    }
    var isValid: Bool { emailError == nil && passwordError == nil }

    func submit() throws -> SignupInput {
        SignupInput(email: try Email(email), password: password)
    }
}

struct SignupView: View {
    @Bindable var form: SignupForm

    var body: some View {
        Form {
            TextField("Email", text: $form.email)
                .onSubmit { form.touched.insert(.email) }
            if form.touched.contains(.email), let e = form.emailError {
                Text(e).foregroundStyle(.red)
            }
            SecureField("Password", text: $form.password)
            Button("Create account") { _ = try? form.submit() }
                .disabled(!form.isValid)
        }
    }
}
```

Equivalences:

- `register("email")` → `$form.email`.
- `formState.errors` → computed properties (`emailError`).
- `isValid` / `isDirty` → computed properties on the model.
- `touched` → a `Set` of fields, or `@FocusState` to know which field has focus.
- The Zod resolver → `submit()` builds validated domain types; if it builds, it is valid.

## Component composition

SwiftUI views are cheap `struct`s; composing is the normal path, as in React.

### Slots / children → `@ViewBuilder`

```swift
struct Card<Header: View, Content: View>: View {
    @ViewBuilder var header: Header
    @ViewBuilder var content: Content

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            header.font(.headline)
            content
        }
        .padding()
        .background(.background.secondary, in: .rect(cornerRadius: 12))
    }
}

Card {
    Text("Recording")
} content: {
    Text("00:42")
}
```

### HOCs / wrappers → `ViewModifier`

```swift
struct LoadingOverlay: ViewModifier {
    let isLoading: Bool
    func body(content: Content) -> some View {
        content.overlay { if isLoading { ProgressView() } }
    }
}

extension View {
    func loading(_ on: Bool) -> some View { modifier(LoadingOverlay(isLoading: on)) }
}

// usage: MyView().loading(store.isLoading)
```

### Context → `@Environment` + `@Entry`

```swift
extension EnvironmentValues {
    @Entry var theme: Theme = .default
}

// provide: .environment(\.theme, .dark)
// consume: @Environment(\.theme) private var theme
```

### Headless components / variants → styles

Style protocols separate behaviour from appearance, like a headless component:

```swift
struct PrimaryButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .padding(.horizontal, 14).padding(.vertical, 8)
            .background(.tint, in: .capsule)
            .foregroundStyle(.white)
            .opacity(configuration.isPressed ? 0.7 : 1)
    }
}

extension ButtonStyle where Self == PrimaryButtonStyle {
    static var primary: Self { .init() }
}

// usage: Button("Save") {}.buttonStyle(.primary)
```

### Other equivalences

- Callback props (`onChange`) → closures: `let onSave: (Item) -> Void`.
- Child → parent without callbacks (sizes, titles) → `PreferenceKey`.
- Generic components → Swift generics (`List<Item: Identifiable>`).
- `useMemo` / `memo` → unnecessary in most cases; extracting small subviews already limits
  re-renders.
- Storybook → `#Preview { }` with mock data.

## Patterns and architecture (DDD / hexagonal)

Swift fits hexagonal well because the boundaries between SwiftPM packages are enforced by the
compiler, not by team discipline. The package layout, the dependency rules and which parts of
hexagonal to keep are in [swift-modular-architecture.md](./swift-modular-architecture.md); this
section only maps the remaining React-side concepts.

### Ports and adapters

```swift
// Domain
public protocol RecordingRepository: Sendable {
    func all() async throws -> [Recording]
    func save(_ r: Recording) async throws
}

// Infrastructure
public struct SQLiteRecordingRepository: RecordingRepository { /* ... */ }

// Tests / previews
struct InMemoryRecordingRepository: RecordingRepository { /* ... */ }
```

### Dependency injection

- Light: pass the port through `init` to the store, or through `Environment`.
- Scalable: `swift-dependencies` (Point-Free), with `@Dependency(\.recordings)` and automatic
  overrides in tests and previews.

```swift
@Observable @MainActor
final class RecordingsStore {
    private let repo: RecordingRepository
    var items: [Recording] = []
    init(repo: RecordingRepository) { self.repo = repo }
    func load() async { items = (try? await repo.all()) ?? [] }
}
```

### Other pieces

| Need                                                     | In Swift                                              |
| -------------------------------------------------------- | ----------------------------------------------------- |
| State shared across threads (recorder, sync, a queue)    | `actor`                                               |
| Concurrency guarantees                                   | Swift 6 strict concurrency: data races at compile time |
| Event streams (RxJS / event emitters)                    | `AsyncStream` / `AsyncSequence`                       |
| Native persistence with little setup                     | SwiftData                                             |
| Explicit SQL, migrations, closest to Drizzle             | GRDB or SQLiteData                                    |
| Secrets and tokens                                       | Keychain                                              |
| Tests                                                    | Swift Testing (`@Test`, `#expect`)                    |
| Typed errors (Result / neverthrow)                       | `throws(MyError)` (typed throws) or `Result`          |

## Practical rules

The typical mistake when coming from React is to replicate the JS stack library by library.
These rules prevent it:

1. Start with only `@Observable`, `Environment`, `.task` and local SwiftPM packages.
2. Validate in `Domain` types, not in views: if an `Email` exists, it is valid.
3. One `@Observable` store per feature, not one giant global store.
4. Views do not know `Infrastructure`; they only see `Domain` protocols.
5. Anything touching threads or shared resources (audio, files, network) lives in an `actor`.
6. Turn on Swift 6 strict concurrency from the start; migrating later hurts more.
7. Add TCA, `swift-dependencies` or `swift-sharing` only when the problem they solve appears.
8. Every view has a `#Preview` fed by in-memory adapters.

## Related

- [swift-modular-architecture.md](./swift-modular-architecture.md) — the package layout these
  patterns live in, and the Swift details that make the layers hold.
- [swift-macos-app-lifecycle.md](./swift-macos-app-lifecycle.md) — where stores and services are
  built (the composition root) versus where side effects are taken (the app delegate).
- [conventions/schemas-first.md](../conventions/schemas-first.md) — the TypeScript side of
  "parse, don't validate".
- [stacks/desktop-swift.md](../stacks/desktop-swift.md) — the stack recipe that links this page.
