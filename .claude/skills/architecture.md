# The Composable Architecture (TCA) - Architecture Guide

> **Version**: 1.18+ (Latest)
> **Last Updated**: 2025-02
> **Scope**: Modern TCA with @Reducer, @ObservableState, @Presents macros

## Overview

The Composable Architecture (TCA) is a library for building applications with a focus on composition, testing, and ergonomics. This guide covers the architectural patterns and best practices for modern TCA development.

---

## Core Concepts

### 1. The Reducer Protocol

Every feature is built as a `@Reducer` struct or enum:

```swift
import ComposableArchitecture

@Reducer
struct Feature {
  @ObservableState
  struct State: Equatable {
    var count = 0
    var isLoading = false
  }

  enum Action {
    case incrementButtonTapped
    case decrementButtonTapped
    case fetchDataButtonTapped
    case dataResponse(Result<Data, Error>)
  }

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .incrementButtonTapped:
        state.count += 1
        return .none

      case .decrementButtonTapped:
        state.count -= 1
        return .none

      case .fetchDataButtonTapped:
        state.isLoading = true
        return .run { send in
          await send(.dataResponse(Result { try await apiClient.fetchData() }))
        }

      case let .dataResponse(.success(data)):
        state.isLoading = false
        // Handle data
        return .none

      case .dataResponse(.failure):
        state.isLoading = false
        return .none
      }
    }
  }
}
```

**Key Principles:**

- **State**: `@ObservableState` struct with all feature data
- **Action**: Enum representing all possible events
- **Reducer body**: Composition point using `Reduce` and other reducers
- **Effects**: Return `Effect<Action>` to handle side effects

---

## State Management

### @ObservableState Macro

**Modern approach** (replaces `@ObservableObject` and `ViewStore`):

```swift
@ObservableState
struct State: Equatable {
  var name = ""
  var email = ""
  var isValid: Bool {
    !name.isEmpty && email.contains("@")
  }
}
```

**Benefits:**
- Fine-grained SwiftUI observation (iOS 17+ / backported to iOS 13+)
- Automatic equality conformance support
- Computed properties work seamlessly
- No need for `ViewStore` in views

### Equatable Conformance

**Always make State Equatable** for:
- TestStore assertions
- Performance optimizations
- Debugging capabilities

```swift
@ObservableState
struct State: Equatable {
  var items: IdentifiedArrayOf<Item> = []
  var selection: UUID?

  // Automatic Equatable synthesis works
}
```

### Shared State

Use `@Shared` for cross-feature state persistence:

```swift
@ObservableState
struct State: Equatable {
  @Shared(.appStorage("hasSeenOnboarding")) var hasSeenOnboarding = false
  @Shared(.fileStorage(.documentsDirectory.appending("data.json")))
  var items: IdentifiedArrayOf<Item> = []
}
```

**Common Strategies:**
- `.appStorage()` - UserDefaults persistence
- `.fileStorage()` - File-based persistence
- `.inMemory()` - In-memory shared state
- Custom `SharedKey` implementations

---

## Action Organization

### Action Categories

Organize actions into semantic categories:

```swift
@Reducer
struct Feature {
  enum Action {
    // User interactions
    case view(ViewAction)

    // Child feature actions
    case destination(PresentationAction<Destination.Action>)
    case path(StackActionOf<Path>)

    // Internal/private actions
    case _internal(InternalAction)

    // Delegate actions (for parent communication)
    case delegate(DelegateAction)

    enum ViewAction {
      case onAppear
      case saveButtonTapped
      case cancelButtonTapped
    }

    enum InternalAction {
      case dataLoadResponse(Result<Data, Error>)
      case timerTicked
    }

    enum DelegateAction {
      case didSave
      case didCancel
    }
  }
}
```

### @ViewAction Macro

Use for compile-time safety in views:

```swift
@Reducer
struct Feature {
  @ObservableState
  struct State { /* ... */ }

  @CasePathable
  enum Action {
    case view(ViewAction)
    // ...

    @CasePathable
    enum ViewAction {
      case saveButtonTapped
      case textChanged(String)
    }
  }
}

// In SwiftUI View:
@ViewAction(for: Feature.self)
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Button("Save") {
      send(.saveButtonTapped) // Automatically wrapped in .view()
    }
  }
}
```

---

## Effect Management

### Modern Effect API

Use `.run` for async effects:

```swift
case .fetchDataButtonTapped:
  return .run { send in
    await send(.dataLoadResponse(Result {
      try await apiClient.fetchData()
    }))
  }
```

### Effect Cancellation

**Named cancellation** for controlling long-running effects:

```swift
private enum CancelID { case timer, request }

case .startTimer:
  return .run { send in
    for await _ in clock.timer(interval: .seconds(1)) {
      await send(.timerTicked)
    }
  }
  .cancellable(id: CancelID.timer)

case .stopTimer:
  return .cancel(id: CancelID.timer)
```

**In-flight cancellation** for debouncing:

```swift
case let .searchQueryChanged(query):
  state.searchQuery = query
  return .run { send in
    try await clock.sleep(for: .milliseconds(300))
    await send(.performSearch(query))
  }
  .cancellable(id: CancelID.search, cancelInFlight: true)
```

### Effect Composition

```swift
return .merge(
  .run { /* effect 1 */ },
  .run { /* effect 2 */ }
)

return .concatenate(
  .run { /* runs first */ },
  .run { /* runs after first completes */ }
)
```

---

## Navigation Patterns

### Tree-Based Navigation (Modals, Sheets, Popovers)

Use `@Presents` macro with destination enum:

```swift
@Reducer
struct Feature {
  @ObservableState
  struct State {
    @Presents var destination: Destination.State?
  }

  enum Action {
    case addButtonTapped
    case destination(PresentationAction<Destination.Action>)
  }

  @Reducer
  enum Destination {
    case add(AddFeature)
    case edit(EditFeature)
    case alert(AlertState<Alert>)

    enum Alert {
      case confirmDelete
    }
  }

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .addButtonTapped:
        state.destination = .add(AddFeature.State())
        return .none

      case .destination(.presented(.add(.delegate(.didSave)))):
        state.destination = nil
        return .none

      case .destination:
        return .none
      }
    }
    .ifLet(\.$destination, action: \.destination)
  }
}

// In View:
struct FeatureView: View {
  @Bindable var store: StoreOf<Feature>

  var body: some View {
    List { /* ... */ }
      .sheet(item: $store.scope(state: \.destination?.add, action: \.destination.add)) { store in
        AddFeatureView(store: store)
      }
      .alert($store.scope(state: \.destination?.alert, action: \.destination.alert))
  }
}
```

### Stack-Based Navigation (NavigationStack)

```swift
@Reducer
struct Feature {
  @ObservableState
  struct State {
    var path = StackState<Path.State>()
  }

  enum Action {
    case path(StackActionOf<Path>)
  }

  @Reducer
  enum Path {
    case detail(DetailFeature)
    case edit(EditFeature)
  }

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      switch action {
      case .path:
        return .none
      }
    }
    .forEach(\.path, action: \.path)
  }
}

// In View:
struct FeatureView: View {
  @Bindable var store: StoreOf<Feature>

  var body: some View {
    NavigationStack(path: $store.scope(state: \.path, action: \.path)) {
      List { /* ... */ }
    } destination: { store in
      switch store.case {
      case let .detail(store):
        DetailFeatureView(store: store)
      case let .edit(store):
        EditFeatureView(store: store)
      }
    }
  }
}
```

### Automatic Effect Cancellation

TCA automatically cancels effects when:
- Destination becomes `nil`
- Stack element is popped
- Enum case changes

**No manual cleanup needed** for navigation-scoped effects!

---

## Dependency Management

### Declaring Dependencies

```swift
@Reducer
struct Feature {
  @Dependency(\.apiClient) var apiClient
  @Dependency(\.continuousClock) var clock
  @Dependency(\.uuid) var uuid

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      case .fetchButtonTapped:
        return .run { send in
          let data = try await apiClient.fetch()
          await send(.dataResponse(data))
        }
    }
  }
}
```

### Registering Dependencies

```swift
struct APIClient {
  var fetch: @Sendable () async throws -> Data
}

extension APIClient: DependencyKey {
  static let liveValue = APIClient(
    fetch: {
      let (data, _) = try await URLSession.shared.data(from: URL(string: "...")!)
      return data
    }
  )
}

extension DependencyValues {
  var apiClient: APIClient {
    get { self[APIClient.self] }
    set { self[APIClient.self] = newValue }
  }
}
```

### Testing with Dependencies

```swift
@Test
func testFetch() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  } withDependencies: {
    $0.apiClient.fetch = { Data("test".utf8) }
  }

  await store.send(.fetchButtonTapped)
  await store.receive(\.dataResponse) {
    $0.data = Data("test".utf8)
  }
}
```

---

## Composition Patterns

### Parent-Child Communication

**Delegate Pattern** for child-to-parent communication:

```swift
@Reducer
struct ChildFeature {
  enum Action {
    case saveButtonTapped
    case delegate(DelegateAction)

    enum DelegateAction {
      case didSave(Item)
    }
  }

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      case .saveButtonTapped:
        return .send(.delegate(.didSave(state.item)))
    }
  }
}

@Reducer
struct ParentFeature {
  enum Action {
    case child(ChildFeature.Action)
  }

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      case .child(.delegate(.didSave(let item))):
        // Handle save in parent
        return .none

      case .child:
        return .none
    }
  }
}
```

### Scoping Stores

```swift
// In parent view:
struct ParentView: View {
  let store: StoreOf<ParentFeature>

  var body: some View {
    if let store = store.scope(state: \.child, action: \.child) {
      ChildView(store: store)
    }
  }
}
```

---

## Testing Architecture

### TestStore Basics

```swift
@Test
func testBasicFlow() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  // Assert state changes
  await store.send(.incrementButtonTapped) {
    $0.count = 1
  }

  // Assert received actions
  await store.receive(\.response) {
    $0.isLoading = false
  }

  // Assert all effects completed
  await store.finish()
}
```

### Non-Exhaustive Testing

For integration tests:

```swift
@Test
func testIntegration() async {
  let store = TestStore(initialState: AppFeature.State()) {
    AppFeature()
  }

  store.exhaustivity = .off

  // Only assert what matters
  await store.send(\.login.submitButtonTapped)
  await store.receive(\.login.delegate.didLogin) {
    $0.isLoggedIn = true
  }
}
```

### Testing Long-Running Effects

```swift
@Test
func testTimer() async {
  let clock = TestClock()

  let store = TestStore(initialState: Feature.State()) {
    Feature()
  } withDependencies: {
    $0.continuousClock = clock
  }

  await store.send(.startTimer)

  await clock.advance(by: .seconds(1))
  await store.receive(\.timerTicked)

  await clock.advance(by: .seconds(1))
  await store.receive(\.timerTicked)

  await store.send(.stopTimer)
}
```

---

## Performance Best Practices

### 1. Use IdentifiedArray for Collections

```swift
@ObservableState
struct State {
  var items: IdentifiedArrayOf<Item> = []
  // NOT: var items: [Item] = []
}
```

**Benefits:**
- O(1) lookup by ID
- Automatic diffing
- Better SwiftUI performance

### 2. Scope Stores Appropriately

```swift
// Good - scoped to specific child
ForEach(store.scope(state: \.items, action: \.items)) { itemStore in
  ItemView(store: itemStore)
}

// Avoid - passing entire parent store
ForEach(items) { item in
  ItemView(store: store, item: item) // Anti-pattern
}
```

### 3. Use Computed Properties for Derived State

```swift
@ObservableState
struct State {
  var items: [Item] = []

  var completedItems: [Item] {
    items.filter(\.isCompleted)
  }

  var progress: Double {
    guard !items.isEmpty else { return 0 }
    return Double(completedItems.count) / Double(items.count)
  }
}
```

### 4. Minimize State Copying

```swift
// Prefer inout mutations
Reduce { state, action in
  state.count += 1
  return .none
}

// Avoid unnecessary copies
// Don't: var newState = state; newState.count += 1; state = newState
```

---

## SwiftUI Integration

### Modern Store Usage (No ViewStore)

```swift
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Form {
      Text("Count: \(store.count)")

      Button("Increment") {
        store.send(.incrementButtonTapped)
      }

      if store.isLoading {
        ProgressView()
      }
    }
  }
}
```

### Bindings

```swift
@Reducer
struct Feature {
  @ObservableState
  struct State {
    var text = ""
    @Presents var destination: Destination.State?
  }

  enum Action: BindableAction {
    case binding(BindingAction<State>)
    case destination(PresentationAction<Destination.Action>)
  }

  var body: some ReducerOf<Self> {
    BindingReducer()
    Reduce { state, action in
      // ...
    }
  }
}

struct FeatureView: View {
  @Bindable var store: StoreOf<Feature>

  var body: some View {
    Form {
      TextField("Text", text: $store.text)

      // For custom bindings
      TextField("Custom", text: $store.text.sending(\.textChanged))
    }
  }
}
```

---

## Migration Notes

### From 1.7 to 1.18+

1. **Replace `@PresentationState` with `@Presents`**:
   ```swift
   // Old
   @PresentationState var destination: Destination.State?

   // New
   @Presents var destination: Destination.State?
   ```

2. **Use `@ObservableState` everywhere**:
   - Removes need for `ViewStore`
   - Better SwiftUI performance
   - Backported to iOS 13+

3. **Prefer `@Shared` over manual synchronization**:
   - Built-in persistence
   - Automatic propagation
   - Type-safe

---

## Common Patterns

### Loading States

```swift
@ObservableState
struct State {
  var data: Data?
  var isLoading = false
  var error: Error?
}

// In reducer
case .loadButtonTapped:
  state.isLoading = true
  state.error = nil
  return .run { send in
    await send(.dataResponse(Result { try await apiClient.fetch() }))
  }

case let .dataResponse(.success(data)):
  state.isLoading = false
  state.data = data
  return .none

case let .dataResponse(.failure(error)):
  state.isLoading = false
  state.error = error
  return .none
```

### Debouncing

```swift
private enum CancelID { case search }

case let .searchQueryChanged(query):
  state.searchQuery = query
  return .run { send in
    try await clock.sleep(for: .milliseconds(300))
    await send(.performSearch(query))
  }
  .cancellable(id: CancelID.search, cancelInFlight: true)
```

### Polling

```swift
case .startPolling:
  return .run { send in
    for await _ in clock.timer(interval: .seconds(5)) {
      await send(.fetchData)
    }
  }
  .cancellable(id: CancelID.polling)
```

---

## Anti-Patterns to Avoid

❌ **Don't mutate state outside of reducers**
❌ **Don't call `store.send()` from effects**
❌ **Don't use `@Published` or `@State` in reducers**
❌ **Don't ignore effects in tests** (use `.finish()`)
❌ **Don't share Store instances across features** (use scoping)
❌ **Don't use global state** (use `@Shared` or dependencies)

---

## Resources

- [Official Documentation](https://swiftpackageindex.com/pointfreeco/swift-composable-architecture/main/documentation/composablearchitecture)
- [Point-Free Episodes](https://www.pointfree.co/collections/composable-architecture)
- [Example Projects](../Examples/)
- [Migration Guides](./Sources/ComposableArchitecture/Documentation.docc/Articles/MigrationGuides/)
