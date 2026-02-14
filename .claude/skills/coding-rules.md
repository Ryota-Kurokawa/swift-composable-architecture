# TCA コーディングルール & ベストプラクティス

> **バージョン**: 1.18+ (最新)
> **最終更新**: 2025-02
> **Swift バージョン**: 6.0+ with strict concurrency

## 目次

1. [ファイル構成](#ファイル構成)
2. [命名規則](#命名規則)
3. [State設計](#state設計)
4. [Action設計](#action設計)
5. [Reducer実装](#reducer実装)
6. [Effectガイドライン](#effectガイドライン)
7. [依存性管理](#依存性管理)
8. [テスト要件](#テスト要件)
9. [SwiftUI統合](#swiftui統合)
10. [コードスタイル](#コードスタイル)

---

## ファイル構成

### 機能モジュール構造

```
FeatureModule/
├── Feature.swift              # メインのReducer
├── FeatureView.swift         # SwiftUI View
├── Models/
│   ├── FeatureModels.swift   # ドメインモデル
│   └── ...
├── Dependencies/
│   ├── APIClient.swift       # 機能の依存性
│   └── ...
└── Tests/
    ├── FeatureTests.swift
    └── ...
```

### 単一ファイル構造

シンプルな機能には単一ファイル構造を使用:

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

## 命名規則

### Reducerの命名

```swift
✅ // 機能ベースの名前
@Reducer struct LoginFeature { }
@Reducer struct ProfileFeature { }
@Reducer struct SettingsFeature { }

❌ // "Reducer" サフィックスを避ける
@Reducer struct LoginReducer { }  // 冗長
```

### Stateの命名

```swift
✅ // 常にReducer内にネスト
@Reducer
struct Feature {
  @ObservableState
  struct State: Equatable {
    var count = 0
  }
}

❌ // State を別に定義しない
struct FeatureState: Equatable { }  // スコープが壊れる
```

### Actionの命名

```swift
✅ // 明確で、過去形のイベント名
enum Action {
  case loginButtonTapped
  case usernameFocused
  case apiResponse(Result<User, Error>)
  case delegate(DelegateAction)
}

❌ // 命令形/曖昧な名前を避ける
enum Action {
  case login          // 不明確
  case tapLoginButton // 冗長
  case onTap          // 汎用的すぎ
}
```

### Viewの命名

```swift
✅ // Reducer名 + "View" に合わせる
struct LoginFeatureView: View { }
struct ProfileFeatureView: View { }

✅ // または曖昧でなければ機能名だけでも可
struct LoginView: View { }

❌ // 規則を混在させない
struct LoginScreen: View { }     // 一貫性がない
struct LoginViewController: View { }  // 間違ったフレームワーク
```

### 依存性の命名

```swift
✅ // サービスには Client/Manager サフィックス
struct APIClient { }
struct DatabaseClient { }
struct LocationManager { }

✅ // コレクション/ユーティリティには複数形
struct DateFormatters { }
struct Validators { }

❌ // "Service" サフィックスを避ける
struct APIService { }  // TCA の慣例ではない
```

---

## State設計

### ルール1: 常に @ObservableState を使用

```swift
✅ // 最新のアプローチ
@ObservableState
struct State: Equatable {
  var name = ""
  var email = ""
}

❌ // マクロを省略しない
struct State: Equatable {  // 監視の利点を失う
  var name = ""
}
```

### ルール2: State を Equatable にする

```swift
✅ // 常に Equatable に準拠
@ObservableState
struct State: Equatable {
  var items: IdentifiedArrayOf<Item> = []
  var selection: UUID?
}

❌ // Equatable がないとテストが壊れる
@ObservableState
struct State {  // TestStore が適切に使えない
  var items: [Item] = []
}
```

### ルール3: 値型を使用

```swift
✅ // 構造体を優先
@ObservableState
struct State: Equatable {
  var user: User
  var settings: Settings
}

❌ // State内で参照型を避ける
@ObservableState
struct State: Equatable {
  var viewModel: ViewModel  // 参照型
}
```

### ルール4: コレクションには IdentifiedArray を使用

```swift
✅ // IdentifiedArray を使用
@ObservableState
struct State: Equatable {
  var items: IdentifiedArrayOf<Item> = []
}

struct Item: Identifiable, Equatable {
  let id: UUID
  var name: String
}

❌ // 識別可能な項目に通常の配列を使わない
@ObservableState
struct State: Equatable {
  var items: [Item] = []  // O(n) 検索、差分計算が悪い
}
```

### ルール5: 派生状態には算出プロパティ

```swift
✅ // 算出プロパティを使用
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

❌ // 派生データを重複させない
@ObservableState
struct State: Equatable {
  var items: [Item] = []
  var completedItems: [Item] = []  // 冗長、同期が取れなくなる可能性
}
```

### ルール6: デフォルト値を提供

```swift
✅ // デフォルト値を提供
@ObservableState
struct State: Equatable {
  var count = 0
  var items: IdentifiedArrayOf<Item> = []
  var isLoading = false
  var error: String?  // nil がデフォルト
}

❌ // 強制アンラップを避ける
@ObservableState
struct State: Equatable {
  var user: User  // 初期化が必要
}
```

### ルール7: ナビゲーションには @Presents を使用

```swift
✅ // @Presents マクロを使用
@ObservableState
struct State: Equatable {
  @Presents var destination: Destination.State?
}

❌ // @PresentationState を使わない（非推奨）
@ObservableState
struct State: Equatable {
  @PresentationState var destination: Destination.State?  // 古いAPI
}
```

### ルール8: 永続化には @Shared を使用

```swift
✅ // 永続的な状態には @Shared を使用
@ObservableState
struct State: Equatable {
  @Shared(.appStorage("hasSeenOnboarding")) var hasSeenOnboarding = false
  @Shared(.fileStorage(.documentsDirectory.appending("data.json")))
  var items: IdentifiedArrayOf<Item> = []
}

❌ // 永続化を手動で管理しない
@ObservableState
struct State: Equatable {
  var hasSeenOnboarding = UserDefaults.standard.bool(forKey: "hasSeenOnboarding")  // アンチパターン
}
```

---

## Action設計

### ルール1: アクションをカテゴリーに整理

```swift
✅ // ネストされた列挙型で整理
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

❌ // すべてのアクションをトップレベルに混在させない
enum Action {
  case onAppear
  case dataLoadResponse(Result<Data, Error>)
  case didSave(Item)
  // 保守が難しい
}
```

### ルール2: @CasePathable を使用

```swift
✅ // パターンマッチングのために @CasePathable を追加
@CasePathable
enum Action {
  case view(ViewAction)
  case delegate(DelegateAction)

  @CasePathable
  enum ViewAction {
    case buttonTapped
  }
}

// これが可能になる:
await store.receive(\.delegate.didSave)
```

### ルール3: レスポンスアクション

```swift
✅ // 非同期レスポンスには Result を含める
enum Action {
  case fetchDataButtonTapped
  case dataResponse(Result<Data, Error>)
}

// Reducerで
case .fetchDataButtonTapped:
  return .run { send in
    await send(.dataResponse(Result {
      try await apiClient.fetchData()
    }))
  }

❌ // 成功/失敗を分けない
enum Action {
  case fetchDataButtonTapped
  case dataResponseSuccess(Data)
  case dataResponseFailure(Error)
}
```

### ルール4: デリゲートパターン

```swift
✅ // 親への通信にはデリゲートを使用
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

❌ // クロージャを使わない
@Reducer
struct ChildFeature {
  var onComplete: () -> Void  // テストできない、シリアライズできない
}
```

### ルール5: バインディングアクション

```swift
✅ // 双方向バインディングには BindingAction を使用
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

❌ // 手動バインディングアクションを作成しない
enum Action {
  case nameChanged(String)      // 手動バインディング
  case emailChanged(String)     // 繰り返し
}
```

---

## Reducer実装

### ルール1: 状態変更には Reduce を使用

```swift
✅ // inout state で Reduce を使用
var body: some ReducerOf<Self> {
  Reduce { state, action in
    switch action {
    case .incrementButtonTapped:
      state.count += 1
      return .none
    }
  }
}

❌ // 新しい状態を返さない
var body: some ReducerOf<Self> {
  Reduce { state, action in
    var newState = state
    newState.count += 1
    state = newState  // 不要
    return .none
  }
}
```

### ルール2: 合成の順序を正しく

```swift
✅ // 正しい合成順序
var body: some ReducerOf<Self> {
  BindingReducer()  // 最初: バインディングを処理

  Reduce { state, action in
    // 2番目: メインロジック
  }
  .ifLet(\.$destination, action: \.destination)  // 3番目: 子Reducer
  .forEach(\.path, action: \.path)

  // 4番目: 副作用（分析、ログ）
  ._printChanges()
}

❌ // 間違った順序
var body: some ReducerOf<Self> {
  Reduce { state, action in
    // 子Reducerがアクションを処理する前に実行される
  }
  .ifLet(\.$destination, action: \.destination)

  BindingReducer()  // 遅すぎる
}
```

### ルール3: すべてのアクションケースを処理

```swift
✅ // すべてのケースを明示的に処理
Reduce { state, action in
  switch action {
  case .view(.onAppear):
    return .run { /* ... */ }

  case .view(.saveButtonTapped):
    return .run { /* ... */ }

  case .delegate:
    return .none  // 親が処理

  case .destination:
    return .none  // 子Reducerが処理
  }
}

❌ // default を使わない
Reduce { state, action in
  switch action {
  case .view(.onAppear):
    return .run { /* ... */ }

  default:  // バグを隠す
    return .none
  }
}
```

### ルール4: エフェクトがない場合は .none を返す

```swift
✅ // 常にエフェクトを返す
case .incrementButtonTapped:
  state.count += 1
  return .none

❌ // return を省略しない
case .incrementButtonTapped:
  state.count += 1
  // return がない（コンパイルエラー）
```

### ルール5: 内部アクションには .send() を使用

```swift
✅ // 即座のアクションには .send() を使用
case .loadData:
  state.isLoading = true
  return .merge(
    .send(.trackAnalytics("load_started")),
    .run { send in
      await send(.dataResponse(Result { try await apiClient.fetch() }))
    }
  )

case .trackAnalytics(let event):
  // 分析を処理
  return .none
```

---

## Effectガイドライン

### ルール1: 非同期エフェクトには .run を使用

```swift
✅ // 最新の async/await アプローチ
case .fetchDataButtonTapped:
  return .run { send in
    await send(.dataResponse(Result {
      try await apiClient.fetchData()
    }))
  }

❌ // パブリッシャーを使わない（レガシー）
case .fetchDataButtonTapped:
  return .publisher {
    apiClient.fetchDataPublisher()
      .map(Action.dataResponseSuccess)
      .catch { .just(.dataResponseFailure($0)) }
  }
```

### ルール2: 状態の値をキャプチャ

```swift
✅ // 不変の状態値をキャプチャ
case .startTimer:
  let interval = state.timerInterval
  return .run { send in
    for await _ in clock.timer(interval: interval) {
      await send(.timerTicked)
    }
  }

❌ // 状態全体をキャプチャしない
case .startTimer:
  return .run { [state] send in  // ミュータブルな状態をキャプチャ
    for await _ in clock.timer(interval: state.timerInterval) {
      await send(.timerTicked)
    }
  }
```

### ルール3: キャンセルIDを使用

```swift
✅ // キャンセルIDを定義
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

❌ // 文字列IDを使わない
case .startTimer:
  return .run { /* ... */ }
    .cancellable(id: "timer")  // 型安全でない
```

### ルール4: デバウンスには cancelInFlight を使用

```swift
✅ // cancelInFlight でデバウンス
case let .searchQueryChanged(query):
  state.searchQuery = query
  return .run { send in
    try await clock.sleep(for: .milliseconds(300))
    await send(.performSearch(query))
  }
  .cancellable(id: CancelID.search, cancelInFlight: true)
```

### ルール5: 並列エフェクトには .merge を使用

```swift
✅ // エフェクトを並列実行
case .onAppear:
  return .merge(
    .run { send in await send(.fetchUser) },
    .run { send in await send(.fetchSettings) },
    .run { send in await send(.trackAnalytics) }
  )
```

### ルール6: 順次エフェクトには .concatenate を使用

```swift
✅ // エフェクトを順次実行
case .submitForm:
  return .concatenate(
    .run { send in await send(.validateForm) },
    .run { send in await send(.uploadData) },
    .run { send in await send(.showSuccess) }
  )
```

### ルール7: エフェクトのエラーを処理

```swift
✅ // 常にエフェクトを Result でラップ
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

❌ // エラーを無視しない
case .fetchDataButtonTapped:
  return .run { send in
    let data = try await apiClient.fetchData()  // クラッシュの可能性
    await send(.dataReceived(data))
  }
```

---

## 依存性管理

### ルール1: @Dependency で依存性を宣言

```swift
✅ // @Dependency プロパティラッパーを使用
@Reducer
struct Feature {
  @Dependency(\.apiClient) var apiClient
  @Dependency(\.continuousClock) var clock
  @Dependency(\.uuid) var uuid
}

❌ // グローバルシングルトンを使わない
@Reducer
struct Feature {
  func reduce(into state: inout State, action: Action) -> Effect<Action> {
    case .fetch:
      return .run { send in
        let data = try await NetworkManager.shared.fetch()  // テストできない
      }
  }
}
```

### ルール2: 依存性を適切に登録

```swift
✅ // 完全な依存性登録
struct APIClient {
  var fetchUser: @Sendable (UUID) async throws -> User
  var updateProfile: @Sendable (User) async throws -> Void
}

extension APIClient: DependencyKey {
  static let liveValue = APIClient(
    fetchUser: { id in
      // 本番実装
    },
    updateProfile: { user in
      // 本番実装
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

### ルール3: クロージャには @Sendable を使用

```swift
✅ // クロージャを @Sendable としてマーク
struct APIClient {
  var fetch: @Sendable () async throws -> Data
}

❌ // @Sendable がない
struct APIClient {
  var fetch: () async throws -> Data  // 並行性警告
}
```

### ルール4: テスト用依存性

```swift
✅ // テスト値を提供
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

// テストで
let store = TestStore(initialState: Feature.State()) {
  Feature()
} withDependencies: {
  $0.apiClient.fetchUser = { _ in User.mock }
}
```

---

## テスト要件

### ルール1: すべての状態変更をテスト

```swift
✅ // すべての状態変更をアサート
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

❌ // アサーションをスキップしない
@Test
func testIncrement() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  await store.send(.incrementButtonTapped)  // アサーションなし
}
```

### ルール2: 受信したすべてのアクションをテスト

```swift
✅ // エフェクトから受信したアクションをアサート
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

❌ // 受信アクションを無視しない
@Test
func testFetch() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  await store.send(.fetchButtonTapped)
  // 欠落: await store.receive(\.dataResponse)
}
```

### ルール3: store.finish() を呼び出す

```swift
✅ // すべてのエフェクトが完了したことを保証
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
  await store.finish()  // タイマーがキャンセルされたことを保証
}
```

### ルール4: 時間ベースのエフェクトには TestClock を使用

```swift
✅ // テストで時間を制御
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

### ルール5: 統合テストには非網羅的モードを使用

```swift
✅ // 統合テストは非網羅的でも可
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

## SwiftUI統合

### ルール1: 直接Storeアクセス（ViewStore不要）

```swift
✅ // @ObservableState での最新アプローチ
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Text("カウント: \(store.count)")
    Button("増加") {
      store.send(.incrementButtonTapped)
    }
  }
}

❌ // ViewStore を使わない
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    WithViewStore(store, observe: { $0 }) { viewStore in  // レガシー
      Text("カウント: \(viewStore.count)")
    }
  }
}
```

### ルール2: バインディングには @Bindable を使用

```swift
✅ // ナビゲーション/バインディングには @Bindable を使用
struct FeatureView: View {
  @Bindable var store: StoreOf<Feature>

  var body: some View {
    TextField("名前", text: $store.name)
      .sheet(item: $store.scope(state: \.destination?.add, action: \.destination.add)) { store in
        AddView(store: store)
      }
  }
}
```

### ルール3: Viewで Store をスコープ

```swift
✅ // 子機能にスコープ
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

❌ // 親Storeを子に渡さない
struct ParentView: View {
  let store: StoreOf<ParentFeature>

  var body: some View {
    ChildView(store: store)  // 間違ったスコープ
  }
}
```

### ルール4: 型安全性のために @ViewAction を使用

```swift
✅ // @ViewAction マクロを使用
@ViewAction(for: Feature.self)
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Button("保存") {
      send(.saveButtonTapped)  // .view() でラップされる
    }
  }
}
```

---

## コードスタイル

### ルール1: MARK コメントを使用

```swift
✅ // MARK で整理
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

### ルール2: アクセス制御

```swift
✅ // 適切なアクセスレベルを使用
@Reducer
public struct Feature {  // モジュール内ならpublic
  @ObservableState
  public struct State: Equatable {  // Public
    public var count = 0  // Public state
    var _internalFlag = false  // Internal state
  }

  public enum Action {  // Public
    case view(ViewAction)
    case _internal(InternalAction)  // アンダースコアのプレフィックス

    public enum ViewAction { }
    enum InternalAction { }  // Internal
  }
}
```

### ルール3: ドキュメント

```swift
✅ // Public APIをドキュメント化
/// ユーザー認証とセッション状態を管理します。
///
/// この機能は、`@Shared` プロパティラッパーを使用して
/// ログイン、ログアウト、セッション永続化を処理します。
@Reducer
public struct AuthenticationFeature {
  /// 現在の認証状態。
  @ObservableState
  public struct State: Equatable {
    /// ユーザーが現在ログインしているかどうか。
    public var isAuthenticated = false

    /// 現在ログインしているユーザー（存在する場合）。
    public var currentUser: User?
  }
}
```

### ルール4: 行の長さ

```swift
✅ // 行を100文字以内に保つ
return .run { send in
  await send(
    .dataResponse(
      Result { try await apiClient.fetchData() }
    )
  )
}

❌ // 長い行を書かない
return .run { send in await send(.dataResponse(Result { try await apiClient.fetchData() })) }
```

### ルール5: トレーリングクロージャ

```swift
✅ // トレーリングクロージャ構文を使用
let store = Store(initialState: Feature.State()) {
  Feature()
}

.run { send in
  await send(.action)
}

❌ // 明示的なクロージャパラメータを使わない
let store = Store(initialState: Feature.State(), reducer: {
  Feature()
})
```

---

## よくある間違い

### ❌ Reducer外で状態を変更

```swift
// 間違い
struct FeatureView: View {
  let store: StoreOf<Feature>

  var body: some View {
    Button("悪い") {
      store.state.count += 1  // コンパイルエラー
    }
  }
}
```

### ❌ エフェクトから send() を呼び出す

```swift
// 間違い
case .action:
  return .run { [store] _ in  // Store をキャプチャしない
    try await clock.sleep(for: .seconds(1))
    store.send(.anotherAction)  // 間違い！
  }

// 正しい
case .action:
  return .run { send in
    try await clock.sleep(for: .seconds(1))
    await send(.anotherAction)  // send パラメータを使用
  }
```

### ❌ グローバル状態を使用

```swift
// 間違い
class GlobalState {
  static let shared = GlobalState()
  var isLoggedIn = false
}

// 正しい - @Shared を使用
@ObservableState
struct State {
  @Shared(.appStorage("isLoggedIn")) var isLoggedIn = false
}
```

### ❌ テストの失敗を無視

```swift
// 間違い
@Test
func test() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  await store.send(.action)
  // 無視: "状態が変更されることを期待しましたが、変更されませんでした"
}

// 正しい
@Test
func test() async {
  let store = TestStore(initialState: Feature.State()) {
    Feature()
  }

  await store.send(.action) {
    $0.count = 1  // 変更をアサート
  }
}
```

---

## チェックリスト

コードを提出する前に確認:

- [ ] すべての `State` 型に `@ObservableState` と `Equatable` がある
- [ ] すべての `Action` 列挙型がカテゴリーに整理されている
- [ ] すべてのエフェクトが `@Sendable` クロージャで `.run` を使用
- [ ] すべての依存性が `@Dependency` プロパティラッパーを使用
- [ ] すべてのテストが状態変更と受信アクションをアサート
- [ ] すべてのテストが `store.finish()` を呼び出す
- [ ] `ViewStore` の使用がない（直接Store アクセスを使用）
- [ ] ナビゲーションが `@Presents` を使用（`@PresentationState` ではない）
- [ ] コレクションが適切な場所で `IdentifiedArray` を使用
- [ ] 強制アンラップや暗黙的アンラップオプショナルがない
- [ ] strict concurrency を有効にしてコンパイル
- [ ] Public API のドキュメント

---

## リソース

- [TCAドキュメント](https://swiftpackageindex.com/pointfreeco/swift-composable-architecture/main/documentation/composablearchitecture)
- [Point-Freeエピソード](https://www.pointfree.co/collections/composable-architecture)
- [マイグレーションガイド](https://github.com/pointfreeco/swift-composable-architecture/tree/main/Sources/ComposableArchitecture/Documentation.docc/Articles/MigrationGuides)
- [Swift スタイルガイド](https://google.github.io/swift/)
