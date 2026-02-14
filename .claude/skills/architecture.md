# The Composable Architecture (TCA) - アーキテクチャガイド

> **バージョン**: 1.18+ (最新)
> **最終更新**: 2025-02
> **対象**: @Reducer, @ObservableState, @Presents マクロを使用した最新TCA

## 概要

The Composable Architecture (TCA) は、合成、テスト、人間工学に重点を置いたアプリケーション構築のためのライブラリです。このガイドでは、最新のTCA開発におけるアーキテクチャパターンとベストプラクティスを説明します。

---

## コアコンセプト

### 1. Reducerプロトコル

すべての機能は `@Reducer` 構造体または列挙型として構築されます:

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
        // データ処理
        return .none

      case .dataResponse(.failure):
        state.isLoading = false
        return .none
      }
    }
  }
}
```

**重要な原則:**

- **State**: `@ObservableState` 構造体ですべての機能データを保持
- **Action**: 発生しうるすべてのイベントを表す列挙型
- **Reducer body**: `Reduce` と他のReducerを使用した合成ポイント
- **Effects**: 副作用を処理するために `Effect<Action>` を返す

---

## 状態管理

### @ObservableState マクロ

**最新のアプローチ** (`@ObservableObject` や `ViewStore` を置き換え):

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

**利点:**
- きめ細かなSwiftUI監視 (iOS 17+ / iOS 13+にバックポート)
- 自動的なEquatable準拠サポート
- 算出プロパティがシームレスに機能
- Viewで `ViewStore` が不要

### Equatable準拠

**常にStateをEquatableにする** 理由:
- TestStoreでのアサーション
- パフォーマンス最適化
- デバッグ機能

```swift
@ObservableState
struct State: Equatable {
  var items: IdentifiedArrayOf<Item> = []
  var selection: UUID?

  // 自動的にEquatableが合成される
}
```

### 共有状態

機能間の状態永続化には `@Shared` を使用:

```swift
@ObservableState
struct State: Equatable {
  @Shared(.appStorage("hasSeenOnboarding")) var hasSeenOnboarding = false
  @Shared(.fileStorage(.documentsDirectory.appending("data.json")))
  var items: IdentifiedArrayOf<Item> = []
}
```

**一般的な戦略:**
- `.appStorage()` - UserDefaults永続化
- `.fileStorage()` - ファイルベース永続化
- `.inMemory()` - インメモリ共有状態
- カスタム `SharedKey` 実装

---

## アクションの構成

### アクションのカテゴリー分け

アクションを意味的なカテゴリーに整理:

```swift
@Reducer
struct Feature {
  enum Action {
    // ユーザーインタラクション
    case view(ViewAction)

    // 子機能のアクション
    case destination(PresentationAction<Destination.Action>)
    case path(StackActionOf<Path>)

    // 内部/プライベートアクション
    case _internal(InternalAction)

    // デリゲートアクション（親への通信用）
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

### @ViewAction マクロ

Viewでのコンパイル時安全性のために使用:

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

// SwiftUI Viewで:
@ViewAction(for: Feature.self)
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Button("保存") {
      send(.saveButtonTapped) // 自動的に .view() でラップされる
    }
  }
}
```

---

## エフェクト管理

### 最新のEffect API

非同期エフェクトには `.run` を使用:

```swift
case .fetchDataButtonTapped:
  return .run { send in
    await send(.dataResponse(Result {
      try await apiClient.fetchData()
    }))
  }
```

### エフェクトのキャンセル

**名前付きキャンセル** で長時間実行エフェクトを制御:

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

**実行中キャンセル** でデバウンス:

```swift
case let .searchQueryChanged(query):
  state.searchQuery = query
  return .run { send in
    try await clock.sleep(for: .milliseconds(300))
    await send(.performSearch(query))
  }
  .cancellable(id: CancelID.search, cancelInFlight: true)
```

### エフェクトの合成

```swift
return .merge(
  .run { /* エフェクト1 */ },
  .run { /* エフェクト2 */ }
)

return .concatenate(
  .run { /* 最初に実行 */ },
  .run { /* 最初の完了後に実行 */ }
)
```

---

## ナビゲーションパターン

### ツリーベースナビゲーション（モーダル、シート、ポップオーバー）

destination列挙型と `@Presents` マクロを使用:

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

// Viewで:
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

### スタックベースナビゲーション（NavigationStack）

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

// Viewで:
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

### 自動エフェクトキャンセル

TCAは以下の場合に自動的にエフェクトをキャンセル:
- Destinationが `nil` になる
- スタック要素がポップされる
- 列挙型のケースが変更される

**ナビゲーションスコープのエフェクトに手動クリーンアップは不要！**

---

## 依存性管理

### 依存性の宣言

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

### 依存性の登録

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

### 依存性を使ったテスト

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

## 合成パターン

### 親子間通信

子から親への通信には **デリゲートパターン** を使用:

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
        // 親で保存を処理
        return .none

      case .child:
        return .none
    }
  }
}
```

### Storeのスコープ

```swift
// 親Viewで:
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

## テストアーキテクチャ

### TestStoreの基本

```swift
@Test
func testBasicFlow() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  // 状態変更をアサート
  await store.send(.incrementButtonTapped) {
    $0.count = 1
  }

  // 受信したアクションをアサート
  await store.receive(\.response) {
    $0.isLoading = false
  }

  // すべてのエフェクトが完了したことをアサート
  await store.finish()
}
```

### 非網羅的テスト

統合テスト用:

```swift
@Test
func testIntegration() async {
  let store = TestStore(initialState: AppFeature.State()) {
    AppFeature()
  }

  store.exhaustivity = .off

  // 重要なことだけをアサート
  await store.send(\.login.submitButtonTapped)
  await store.receive(\.login.delegate.didLogin) {
    $0.isLoggedIn = true
  }
}
```

### 長時間実行エフェクトのテスト

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

## パフォーマンスベストプラクティス

### 1. コレクションにはIdentifiedArrayを使用

```swift
@ObservableState
struct State {
  var items: IdentifiedArrayOf<Item> = []
  // 非推奨: var items: [Item] = []
}
```

**利点:**
- IDでのO(1)検索
- 自動差分計算
- 優れたSwiftUIパフォーマンス

### 2. Storeを適切にスコープ

```swift
// 良い - 特定の子にスコープ
ForEach(store.scope(state: \.items, action: \.items)) { itemStore in
  ItemView(store: itemStore)
}

// 避ける - 親Store全体を渡す
ForEach(items) { item in
  ItemView(store: store, item: item) // アンチパターン
}
```

### 3. 派生状態には算出プロパティを使用

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

### 4. 状態のコピーを最小化

```swift
// inout変更を推奨
Reduce { state, action in
  state.count += 1
  return .none
}

// 不要なコピーを避ける
// 非推奨: var newState = state; newState.count += 1; state = newState
```

---

## SwiftUI統合

### 最新のStore使用法（ViewStore不要）

```swift
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Form {
      Text("カウント: \(store.count)")

      Button("増加") {
        store.send(.incrementButtonTapped)
      }

      if store.isLoading {
        ProgressView()
      }
    }
  }
}
```

### バインディング

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
      TextField("テキスト", text: $store.text)

      // カスタムバインディング用
      TextField("カスタム", text: $store.text.sending(\.textChanged))
    }
  }
}
```

---

## マイグレーションノート

### 1.7から1.18+へ

1. **`@PresentationState` を `@Presents` に置き換え**:
   ```swift
   // 旧
   @PresentationState var destination: Destination.State?

   // 新
   @Presents var destination: Destination.State?
   ```

2. **すべての場所で `@ObservableState` を使用**:
   - `ViewStore` が不要に
   - SwiftUIパフォーマンスの向上
   - iOS 13+にバックポート

3. **手動同期より `@Shared` を優先**:
   - 組み込みの永続化
   - 自動伝播
   - 型安全

---

## 一般的なパターン

### ローディング状態

```swift
@ObservableState
struct State {
  var data: Data?
  var isLoading = false
  var error: Error?
}

// Reducerで
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

### デバウンス

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

### ポーリング

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

## 避けるべきアンチパターン

❌ **Reducer外で状態を変更しない**
❌ **エフェクトから `store.send()` を呼び出さない**
❌ **Reducerで `@Published` や `@State` を使用しない**
❌ **テストでエフェクトを無視しない**（`.finish()` を使用）
❌ **機能間でStoreインスタンスを共有しない**（スコープを使用）
❌ **グローバル状態を使用しない**（`@Shared` または依存性を使用）

---

## リソース

- [公式ドキュメント](https://swiftpackageindex.com/pointfreeco/swift-composable-architecture/main/documentation/composablearchitecture)
- [Point-Freeエピソード](https://www.pointfree.co/collections/composable-architecture)
- [サンプルプロジェクト](../Examples/)
- [マイグレーションガイド](./Sources/ComposableArchitecture/Documentation.docc/Articles/MigrationGuides/)
