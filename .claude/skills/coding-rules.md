# TCA Coding Rules & Best Practices

> **Version**: 1.18+ (Latest)
> **Last Updated**: 2025-02
> **Swift Version**: 6.0+ with strict concurrency

## Table of Contents

1. [File Organization](#file-organization)
2. [Naming Conventions](#naming-conventions)
3. [State Design](#state-design)
4. [Action Design](#action-design)
5. [Reducer Implementation](#reducer-implementation)
6. [Effect Guidelines](#effect-guidelines)
7. [Dependency Management](#dependency-management)
8. [Testing Requirements](#testing-requirements)
9. [SwiftUI Integration](#swiftui-integration)
10. [Code Style](#code-style)

---

## File Organization

### Feature Module Structure

```
FeatureModule/
├── Feature.swift              # Main reducer
├── FeatureView.swift         # SwiftUI view
├── Models/
│   ├── FeatureModels.swift   # Domain models
│   └── ...
├── Dependencies/
│   ├── APIClient.swift       # Feature dependencies
│   └── ...
└── Tests/
    ├── FeatureTests.swift
    └── ...
```

### Single File Structure

For simple features, use single-file structure:

```swift
import ComposableArchitecture
import SwiftUI

// MARK: - Feature

@Reducer
struct Feature {
  // MARK: State

  @ObservableState
  struct State: Equatable {
    // ...
  }

  // MARK: Action

  enum Action {
    // ...
  }

  // MARK: Dependencies

  @Dependency(\.apiClient) var apiClient
  @Dependency(\.continuousClock) var clock

  // MARK: Body

  var body: some ReducerOf<Self> {
    Reduce { state, action in
      // ...
    }
  }
}

// MARK: - View

struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    // ...
  }
}

// MARK: - Previews

#Preview {
  FeatureView(
    store: Store(initialState: Feature.State()) {
      Feature()
    }
  )
}
```

---

## Naming Conventions

### Reducer Naming

```swift
✅ // Feature-based names
@Reducer struct LoginFeature { }
@Reducer struct ProfileFeature { }
@Reducer struct SettingsFeature { }

❌ // Avoid "Reducer" suffix
@Reducer struct LoginReducer { }  // Redundant
```

### State Naming

```swift
✅ // Always nested in reducer
@Reducer
struct Feature {
  @ObservableState
  struct State: Equatable {
    var count = 0
  }
}

❌ // Don't define State separately
struct FeatureState: Equatable { }  // Breaks scoping
```

### Action Naming

```swift
✅ // Clear, past-tense for events
enum Action {
  case loginButtonTapped
  case usernameFocused
  case apiResponse(Result<User, Error>)
  case delegate(DelegateAction)
}

❌ // Avoid imperative/ambiguous names
enum Action {
  case login          // Unclear
  case tapLoginButton // Verbose
  case onTap          // Too generic
}
```

### View Naming

```swift
✅ // Match reducer name + "View"
struct LoginFeatureView: View { }
struct ProfileFeatureView: View { }

✅ // Or just feature name if no ambiguity
struct LoginView: View { }

❌ // Don't mix conventions
struct LoginScreen: View { }     // Inconsistent
struct LoginViewController: View { }  // Wrong framework
```

### Dependency Naming

```swift
✅ // Client/Manager suffix for services
struct APIClient { }
struct DatabaseClient { }
struct LocationManager { }

✅ // Plural for collections/utilities
struct DateFormatters { }
struct Validators { }

❌ // Avoid "Service" suffix
struct APIService { }  // Not TCA convention
```

---

## State Design

### Rule 1: Always Use @ObservableState

```swift
✅ // Modern approach
@ObservableState
struct State: Equatable {
  var name = ""
  var email = ""
}

❌ // Don't omit the macro
struct State: Equatable {  // Loses observation benefits
  var name = ""
}
```

### Rule 2: Make State Equatable

```swift
✅ // Always conform to Equatable
@ObservableState
struct State: Equatable {
  var items: IdentifiedArrayOf<Item> = []
  var selection: UUID?
}

❌ // Missing Equatable breaks testing
@ObservableState
struct State {  // Can't use TestStore properly
  var items: [Item] = []
}
```

### Rule 3: Use Value Types

```swift
✅ // Prefer structs
@ObservableState
struct State: Equatable {
  var user: User
  var settings: Settings
}

❌ // Avoid reference types in state
@ObservableState
struct State: Equatable {
  var viewModel: ViewModel  // Reference type
}
```

### Rule 4: Use IdentifiedArray for Collections

```swift
✅ // Use IdentifiedArray
@ObservableState
struct State: Equatable {
  var items: IdentifiedArrayOf<Item> = []
}

struct Item: Identifiable, Equatable {
  let id: UUID
  var name: String
}

❌ // Don't use plain arrays for identified items
@ObservableState
struct State: Equatable {
  var items: [Item] = []  // O(n) lookups, poor diffing
}
```

### Rule 5: Computed Properties for Derived State

```swift
✅ // Use computed properties
@ObservableState
struct State: Equatable {
  var items: [Item] = []

  var completedItems: [Item] {
    items.filter(\.isCompleted)
  }

  var progress: Double {
    guard !items.isEmpty else { return 0 }
    return Double(completedItems.count) / Double(items.count)
  }
}

❌ // Don't duplicate derived data
@ObservableState
struct State: Equatable {
  var items: [Item] = []
  var completedItems: [Item] = []  // Redundant, can desync
}
```

### Rule 6: Default Values

```swift
✅ // Provide default values
@ObservableState
struct State: Equatable {
  var count = 0
  var items: IdentifiedArrayOf<Item> = []
  var isLoading = false
  var error: String?  // nil is the default
}

❌ // Avoid force-unwrapping
@ObservableState
struct State: Equatable {
  var user: User  // Must be initialized
}
```

### Rule 7: Use @Presents for Navigation

```swift
✅ // Use @Presents macro
@ObservableState
struct State: Equatable {
  @Presents var destination: Destination.State?
}

❌ // Don't use @PresentationState (deprecated)
@ObservableState
struct State: Equatable {
  @PresentationState var destination: Destination.State?  // Old API
}
```

### Rule 8: Use @Shared for Persistence

```swift
✅ // Use @Shared for persistent state
@ObservableState
struct State: Equatable {
  @Shared(.appStorage("hasSeenOnboarding")) var hasSeenOnboarding = false
  @Shared(.fileStorage(.documentsDirectory.appending("data.json")))
  var items: IdentifiedArrayOf<Item> = []
}

❌ // Don't manage persistence manually
@ObservableState
struct State: Equatable {
  var hasSeenOnboarding = UserDefaults.standard.bool(forKey: "hasSeenOnboarding")  // Anti-pattern
}
```

---

## Action Design

### Rule 1: Organize Actions into Categories

```swift
✅ // Use nested enums for organization
enum Action {
  case view(ViewAction)
  case `internal`(InternalAction)
  case delegate(DelegateAction)
  case destination(PresentationAction<Destination.Action>)

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
    case didSave(Item)
    case didCancel
  }
}

❌ // Don't mix all actions at top level
enum Action {
  case onAppear
  case dataLoadResponse(Result<Data, Error>)
  case didSave(Item)
  // Hard to maintain
}
```

### Rule 2: Use @CasePathable

```swift
✅ // Add @CasePathable for pattern matching
@CasePathable
enum Action {
  case view(ViewAction)
  case delegate(DelegateAction)

  @CasePathable
  enum ViewAction {
    case buttonTapped
  }
}

// Enables:
await store.receive(\.delegate.didSave)
```

### Rule 3: Response Actions

```swift
✅ // Include Result for async responses
enum Action {
  case fetchDataButtonTapped
  case dataResponse(Result<Data, Error>)
}

// In reducer
case .fetchDataButtonTapped:
  return .run { send in
    await send(.dataResponse(Result {
      try await apiClient.fetchData()
    }))
  }

❌ // Don't split success/failure
enum Action {
  case fetchDataButtonTapped
  case dataResponseSuccess(Data)
  case dataResponseFailure(Error)
}
```

### Rule 4: Delegate Pattern

```swift
✅ // Use delegate for parent communication
@Reducer
struct ChildFeature {
  enum Action {
    case delegate(DelegateAction)

    enum DelegateAction: Equatable {
      case didComplete
      case didCancel
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

❌ // Don't use closures
@Reducer
struct ChildFeature {
  var onComplete: () -> Void  // Not testable, not serializable
}
```

### Rule 5: Binding Actions

```swift
✅ // Use BindingAction for two-way bindings
enum Action: BindableAction {
  case binding(BindingAction<State>)
  case view(ViewAction)
}

var body: some ReducerOf<Self> {
  BindingReducer()
  Reduce { state, action in
    // ...
  }
}

❌ // Don't create manual binding actions
enum Action {
  case nameChanged(String)      // Manual binding
  case emailChanged(String)     // Repetitive
}
```

---

## Reducer Implementation

### Rule 1: Use Reduce for State Mutations

```swift
✅ // Use Reduce with inout state
var body: some ReducerOf<Self> {
  Reduce { state, action in
    switch action {
    case .incrementButtonTapped:
      state.count += 1
      return .none
    }
  }
}

❌ // Don't return new state
var body: some ReducerOf<Self> {
  Reduce { state, action in
    var newState = state
    newState.count += 1
    state = newState  // Unnecessary
    return .none
  }
}
```

### Rule 2: Order Composition Correctly

```swift
✅ // Correct composition order
var body: some ReducerOf<Self> {
  BindingReducer()  // First: handle bindings

  Reduce { state, action in
    // Second: main logic
  }
  .ifLet(\.$destination, action: \.destination)  // Third: child reducers
  .forEach(\.path, action: \.path)

  // Fourth: side effects (analytics, logging)
  ._printChanges()
}

❌ // Wrong order
var body: some ReducerOf<Self> {
  Reduce { state, action in
    // Will run before child reducers handle actions
  }
  .ifLet(\.$destination, action: \.destination)

  BindingReducer()  // Too late
}
```

### Rule 3: Handle All Action Cases

```swift
✅ // Handle all cases explicitly
Reduce { state, action in
  switch action {
  case .view(.onAppear):
    return .run { /* ... */ }

  case .view(.saveButtonTapped):
    return .run { /* ... */ }

  case .delegate:
    return .none  // Parent handles this

  case .destination:
    return .none  // Child reducer handles this
  }
}

❌ // Don't use default
Reduce { state, action in
  switch action {
  case .view(.onAppear):
    return .run { /* ... */ }

  default:  // Hides bugs
    return .none
  }
}
```

### Rule 4: Return .none When No Effects

```swift
✅ // Always return an effect
case .incrementButtonTapped:
  state.count += 1
  return .none

❌ // Don't omit return
case .incrementButtonTapped:
  state.count += 1
  // Missing return (compile error)
```

### Rule 5: Use .send() for Internal Actions

```swift
✅ // Use .send() for immediate actions
case .loadData:
  state.isLoading = true
  return .merge(
    .send(.trackAnalytics("load_started")),
    .run { send in
      await send(.dataResponse(Result { try await apiClient.fetch() }))
    }
  )

case .trackAnalytics(let event):
  // Handle analytics
  return .none
```

---

## Effect Guidelines

### Rule 1: Use .run for Async Effects

```swift
✅ // Modern async/await approach
case .fetchDataButtonTapped:
  return .run { send in
    await send(.dataResponse(Result {
      try await apiClient.fetchData()
    }))
  }

❌ // Don't use publishers (legacy)
case .fetchDataButtonTapped:
  return .publisher {
    apiClient.fetchDataPublisher()
      .map(Action.dataResponseSuccess)
      .catch { .just(.dataResponseFailure($0)) }
  }
```

### Rule 2: Capture State Values

```swift
✅ // Capture immutable state values
case .startTimer:
  let interval = state.timerInterval
  return .run { send in
    for await _ in clock.timer(interval: interval) {
      await send(.timerTicked)
    }
  }

❌ // Don't capture entire state
case .startTimer:
  return .run { [state] send in  // Captures mutable state
    for await _ in clock.timer(interval: state.timerInterval) {
      await send(.timerTicked)
    }
  }
```

### Rule 3: Use Cancellation IDs

```swift
✅ // Define cancellation IDs
private enum CancelID {
  case timer
  case request
  case search
}

case .startTimer:
  return .run { send in
    for await _ in clock.timer(interval: .seconds(1)) {
      await send(.timerTicked)
    }
  }
  .cancellable(id: CancelID.timer)

case .stopTimer:
  return .cancel(id: CancelID.timer)

❌ // Don't use string IDs
case .startTimer:
  return .run { /* ... */ }
    .cancellable(id: "timer")  // Type-unsafe
```

### Rule 4: Use cancelInFlight for Debouncing

```swift
✅ // Debounce with cancelInFlight
case let .searchQueryChanged(query):
  state.searchQuery = query
  return .run { send in
    try await clock.sleep(for: .milliseconds(300))
    await send(.performSearch(query))
  }
  .cancellable(id: CancelID.search, cancelInFlight: true)
```

### Rule 5: Use .merge for Parallel Effects

```swift
✅ // Run effects in parallel
case .onAppear:
  return .merge(
    .run { send in await send(.fetchUser) },
    .run { send in await send(.fetchSettings) },
    .run { send in await send(.trackAnalytics) }
  )
```

### Rule 6: Use .concatenate for Sequential Effects

```swift
✅ // Run effects sequentially
case .submitForm:
  return .concatenate(
    .run { send in await send(.validateForm) },
    .run { send in await send(.uploadData) },
    .run { send in await send(.showSuccess) }
  )
```

### Rule 7: Handle Effect Errors

```swift
✅ // Always wrap effects in Result
case .fetchDataButtonTapped:
  state.isLoading = true
  return .run { send in
    await send(.dataResponse(Result {
      try await apiClient.fetchData()
    }))
  }

case let .dataResponse(.success(data)):
  state.isLoading = false
  state.data = data
  return .none

case let .dataResponse(.failure(error)):
  state.isLoading = false
  state.error = error
  return .none

❌ // Don't ignore errors
case .fetchDataButtonTapped:
  return .run { send in
    let data = try await apiClient.fetchData()  // Can crash
    await send(.dataReceived(data))
  }
```

---

## Dependency Management

### Rule 1: Declare Dependencies with @Dependency

```swift
✅ // Use @Dependency property wrapper
@Reducer
struct Feature {
  @Dependency(\.apiClient) var apiClient
  @Dependency(\.continuousClock) var clock
  @Dependency(\.uuid) var uuid
}

❌ // Don't use global singletons
@Reducer
struct Feature {
  func reduce(into state: inout State, action: Action) -> Effect<Action> {
    case .fetch:
      return .run { send in
        let data = try await NetworkManager.shared.fetch()  // Not testable
      }
  }
}
```

### Rule 2: Register Dependencies Properly

```swift
✅ // Complete dependency registration
struct APIClient {
  var fetchUser: @Sendable (UUID) async throws -> User
  var updateProfile: @Sendable (User) async throws -> Void
}

extension APIClient: DependencyKey {
  static let liveValue = APIClient(
    fetchUser: { id in
      // Live implementation
    },
    updateProfile: { user in
      // Live implementation
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

### Rule 3: Use @Sendable for Closures

```swift
✅ // Mark closures as @Sendable
struct APIClient {
  var fetch: @Sendable () async throws -> Data
}

❌ // Missing @Sendable
struct APIClient {
  var fetch: () async throws -> Data  // Concurrency warning
}
```

### Rule 4: Test Dependencies

```swift
✅ // Provide test values
extension APIClient {
  static let testValue = APIClient(
    fetchUser: { _ in
      throw DependencyError.testValue
    },
    updateProfile: { _ in
      throw DependencyError.testValue
    }
  )
}

// In tests
let store = TestStore(initialState: Feature.State()) {
  Feature()
} withDependencies: {
  $0.apiClient.fetchUser = { _ in User.mock }
}
```

---

## Testing Requirements

### Rule 1: Test All State Changes

```swift
✅ // Assert every state change
@Test
func testIncrement() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  await store.send(.incrementButtonTapped) {
    $0.count = 1
  }

  await store.send(.incrementButtonTapped) {
    $0.count = 2
  }
}

❌ // Don't skip assertions
@Test
func testIncrement() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  await store.send(.incrementButtonTapped)  // No assertion
}
```

### Rule 2: Test All Received Actions

```swift
✅ // Assert received actions from effects
@Test
func testFetch() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  } withDependencies: {
    $0.apiClient.fetch = { Data("test".utf8) }
  }

  await store.send(.fetchButtonTapped) {
    $0.isLoading = true
  }

  await store.receive(\.dataResponse.success) {
    $0.isLoading = false
    $0.data = Data("test".utf8)
  }
}

❌ // Don't ignore received actions
@Test
func testFetch() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  await store.send(.fetchButtonTapped)
  // Missing: await store.receive(\.dataResponse)
}
```

### Rule 3: Call store.finish()

```swift
✅ // Ensure all effects complete
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

  await store.send(.stopTimer)
  await store.finish()  // Ensures timer cancelled
}
```

### Rule 4: Use TestClock for Time-Based Effects

```swift
✅ // Control time in tests
@Test
func testDebounce() async {
  let clock = TestClock()
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  } withDependencies: {
    $0.continuousClock = clock
  }

  await store.send(.searchQueryChanged("a"))
  await clock.advance(by: .milliseconds(200))
  await store.send(.searchQueryChanged("ab"))

  await clock.advance(by: .milliseconds(300))
  await store.receive(\.performSearch) {
    $0.searchQuery = "ab"
  }
}
```

### Rule 5: Use Non-Exhaustive for Integration Tests

```swift
✅ // Integration tests can be non-exhaustive
@Test
func testLoginFlow() async {
  let store = TestStore(initialState: AppFeature.State()) {
    AppFeature()
  }

  store.exhaustivity = .off

  await store.send(\.login.submitButtonTapped)
  await store.receive(\.login.delegate.didLogin) {
    $0.isLoggedIn = true
  }
}
```

---

## SwiftUI Integration

### Rule 1: Direct Store Access (No ViewStore)

```swift
✅ // Modern approach with @ObservableState
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Text("Count: \(store.count)")
    Button("Increment") {
      store.send(.incrementButtonTapped)
    }
  }
}

❌ // Don't use ViewStore
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    WithViewStore(store, observe: { $0 }) { viewStore in  // Legacy
      Text("Count: \(viewStore.count)")
    }
  }
}
```

### Rule 2: Use @Bindable for Bindings

```swift
✅ // Use @Bindable for navigation/bindings
struct FeatureView: View {
  @Bindable var store: StoreOf<Feature>

  var body: some View {
    TextField("Name", text: $store.name)
      .sheet(item: $store.scope(state: \.destination?.add, action: \.destination.add)) { store in
        AddView(store: store)
      }
  }
}
```

### Rule 3: Scope Stores in Views

```swift
✅ // Scope to child features
struct ParentView: View {
  let store: StoreOf<ParentFeature>

  var body: some View {
    VStack {
      if let childStore = store.scope(state: \.child, action: \.child) {
        ChildView(store: childStore)
      }
    }
  }
}

❌ // Don't pass parent store to children
struct ParentView: View {
  let store: StoreOf<ParentFeature>

  var body: some View {
    ChildView(store: store)  // Wrong scope
  }
}
```

### Rule 4: Use @ViewAction for Type Safety

```swift
✅ // Use @ViewAction macro
@ViewAction(for: Feature.self)
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Button("Save") {
      send(.saveButtonTapped)  // Wrapped in .view()
    }
  }
}
```

---

## Code Style

### Rule 1: Use MARK Comments

```swift
✅ // Organize with MARK
@Reducer
struct Feature {
  // MARK: - State

  @ObservableState
  struct State: Equatable { }

  // MARK: - Action

  enum Action { }

  // MARK: - Dependencies

  @Dependency(\.apiClient) var apiClient

  // MARK: - Reducer

  var body: some ReducerOf<Self> { }
}
```

### Rule 2: Access Control

```swift
✅ // Use appropriate access levels
@Reducer
public struct Feature {  // Public if in module
  @ObservableState
  public struct State: Equatable {  // Public
    public var count = 0  // Public state
    var _internalFlag = false  // Internal state
  }

  public enum Action {  // Public
    case view(ViewAction)
    case _internal(InternalAction)  // Prefix with underscore

    public enum ViewAction { }
    enum InternalAction { }  // Internal
  }
}
```

### Rule 3: Documentation

```swift
✅ // Document public APIs
/// Manages user authentication and session state.
///
/// This feature handles login, logout, and session persistence
/// using the `@Shared` property wrapper for state synchronization.
@Reducer
public struct AuthenticationFeature {
  /// The current authentication state.
  @ObservableState
  public struct State: Equatable {
    /// Whether a user is currently logged in.
    public var isAuthenticated = false

    /// The currently logged-in user, if any.
    public var currentUser: User?
  }
}
```

### Rule 4: Line Length

```swift
✅ // Keep lines under 100 characters
return .run { send in
  await send(
    .dataResponse(
      Result { try await apiClient.fetchData() }
    )
  )
}

❌ // Don't write long lines
return .run { send in await send(.dataResponse(Result { try await apiClient.fetchData() })) }
```

### Rule 5: Trailing Closures

```swift
✅ // Use trailing closure syntax
let store = Store(initialState: Feature.State()) {
  Feature()
}

.run { send in
  await send(.action)
}

❌ // Don't use explicit closure parameter
let store = Store(initialState: Feature.State(), reducer: {
  Feature()
})
```

---

## Common Mistakes

### ❌ Mutating State Outside Reducers

```swift
// WRONG
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Button("Bad") {
      store.state.count += 1  // Compile error
    }
  }
}
```

### ❌ Calling send() from Effects

```swift
// WRONG
case .action:
  return .run { [store] _ in  // Don't capture store
    try await clock.sleep(for: .seconds(1))
    store.send(.anotherAction)  // Wrong!
  }

// CORRECT
case .action:
  return .run { send in
    try await clock.sleep(for: .seconds(1))
    await send(.anotherAction)  // Use the send parameter
  }
```

### ❌ Using Global State

```swift
// WRONG
class GlobalState {
  static let shared = GlobalState()
  var isLoggedIn = false
}

// CORRECT - Use @Shared
@ObservableState
struct State {
  @Shared(.appStorage("isLoggedIn")) var isLoggedIn = false
}
```

### ❌ Ignoring Test Failures

```swift
// WRONG
@Test
func test() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  await store.send(.action)
  // Ignoring: "Expected state to change but it didn't"
}

// CORRECT
@Test
func test() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  await store.send(.action) {
    $0.count = 1  // Assert the change
  }
}
```

---

## Checklist

Before submitting code, verify:

- [ ] All `State` types have `@ObservableState` and `Equatable`
- [ ] All `Action` enums are organized into categories
- [ ] All effects use `.run` with `@Sendable` closures
- [ ] All dependencies use `@Dependency` property wrapper
- [ ] All tests assert state changes and received actions
- [ ] All tests call `store.finish()`
- [ ] No `ViewStore` usage (use direct store access)
- [ ] Navigation uses `@Presents` (not `@PresentationState`)
- [ ] Collections use `IdentifiedArray` where appropriate
- [ ] No force-unwrapping or implicitly unwrapped optionals
- [ ] Code compiles with strict concurrency enabled
- [ ] Documentation for public APIs

---

## Resources

- [TCA Documentation](https://swiftpackageindex.com/pointfreeco/swift-composable-architecture/main/documentation/composablearchitecture)
- [Point-Free Episodes](https://www.pointfree.co/collections/composable-architecture)
- [Migration Guides](https://github.com/pointfreeco/swift-composable-architecture/tree/main/Sources/ComposableArchitecture/Documentation.docc/Articles/MigrationGuides)
- [Swift Style Guide](https://google.github.io/swift/)
