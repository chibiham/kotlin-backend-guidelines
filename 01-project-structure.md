# プロジェクト構成とアーキテクチャ

## 技術スタック

- Kotlin
- Spring Boot
- Spring WebFlux（リアクティブスタック）
- R2DBC（リアクティブデータベースアクセス）

---

## パッケージ構成

レイヤー単位でトップレベルを分割し、各レイヤー内で機能別にサブパッケージを構成する。

```
com.example.myapp/
├── MyAppApplication.kt
├── config/                             # アプリケーション全体の設定
│   ├── SecurityConfig.kt
│   ├── WebFluxConfig.kt
│   └── R2dbcConfig.kt
├── common/                             # 共通コンポーネント
│   ├── exception/
│   │   ├── ApplicationException.kt
│   │   └── GlobalExceptionHandler.kt
│   └── extension/
│       └── ...
├── domain/                             # ドメイン層
│   ├── user/
│   │   ├── User.kt                     # エンティティ（@Table付き）
│   │   ├── UserId.kt                   # Value Object
│   │   ├── Email.kt
│   │   ├── UserName.kt
│   │   └── UserRepository.kt           # リポジトリインターフェース
│   └── order/
│       ├── Order.kt
│       └── OrderRepository.kt
├── application/                        # アプリケーション層
│   ├── user/
│   │   ├── UserService.kt
│   │   └── CreateUserCommand.kt
│   └── order/
│       └── OrderService.kt
├── infrastructure/                     # インフラ層
│   ├── user/
│   │   └── UserRepositoryImpl.kt
│   └── order/
│       └── OrderRepositoryImpl.kt
└── presentation/                       # プレゼンテーション層
    ├── user/
    │   ├── UserHandler.kt
    │   ├── UserRouter.kt
    │   └── dto/
    │       ├── UserRequest.kt
    │       └── UserResponse.kt
    └── order/
        └── ...
```

### この構成を採用する理由

- レイヤー間の依存関係がパッケージ構造で明確になる
- 小〜中規模プロジェクトではファイル数が少なく見通しが良い
- レイヤーごとの責務が視覚的に把握しやすい

### 将来的な拡張

プロジェクトが大規模化した場合、機能別パッケージ（Package by Feature）への移行を検討する。

```
# 大規模化時: 機能別パッケージ
com.example.myapp/
├── user/
│   ├── domain/
│   ├── application/
│   ├── infrastructure/
│   └── presentation/
└── order/
    └── ...
```

移行の判断基準：

- 1 つの機能に関わるファイルが多くなり、横断的な変更が頻発する場合
- 機能単位でのコードオーナーシップを明確にしたい場合
- マイクロサービス化を見据える場合

---

## レイヤー構成と依存関係

### 依存関係の方向

```
┌─────────────────┐     ┌─────────────────┐
│  presentation   │────▶│   application   │
└─────────────────┘     └────────┬────────┘
                                 │
                                 ▼
┌─────────────────┐     ┌─────────────────┐
│ infrastructure  │────▶│     domain      │
└─────────────────┘     └─────────────────┘
```

| レイヤー       | 依存先              |
| -------------- | ------------------- |
| domain         | なし                |
| application    | domain のみ         |
| infrastructure | application, domain |
| presentation   | application, domain |

### 各レイヤーの責務

| レイヤー       | 責務                 | 配置するもの                                                |
| -------------- | -------------------- | ----------------------------------------------------------- |
| domain         | ビジネスルールの表現 | エンティティ、Value Object、リポジトリ IF、ドメインサービス |
| application    | ユースケースの実装   | アプリケーションサービス、コマンド/クエリ                   |
| infrastructure | 技術的関心事の実装   | リポジトリ実装、外部 API 連携                               |
| presentation   | HTTP 層の処理        | Router、Handler、DTO                                        |

---

## ドメイン層

### エンティティ

エンティティは DB マッピングアノテーションを持つが、ビジネスロジックは純粋に保つ。

```kotlin
// domain/User.kt
@Table("users")
data class User private constructor(
    @Id val id: UserId,
    val email: Email,
    val name: UserName,
    val status: UserStatus,
    val createdAt: Instant
) {
    companion object {
        // 新規作成用ファクトリ
        fun create(email: Email, name: UserName): User {
            return User(
                id = UserId.generate(),
                email = email,
                name = name,
                status = UserStatus.PENDING,
                createdAt = Instant.now()
            )
        }

        // DB復元用ファクトリ
        fun reconstruct(
            id: UserId,
            email: Email,
            name: UserName,
            status: UserStatus,
            createdAt: Instant
        ): User = User(id, email, name, status, createdAt)
    }

    // 状態遷移メソッド
    fun activate(): User {
        check(status == UserStatus.PENDING) { "PENDINGのユーザーのみ有効化できます" }
        return copy(status = UserStatus.ACTIVE)
    }
}
```

### Value Object

値の妥当性をコンストラクタで保証する。

```kotlin
// domain/UserId.kt
@JvmInline
value class UserId(val value: String) {
    companion object {
        fun generate(): UserId {
            // UUID v7を使用（時系列ソート可能、インデックス効率良好）
            val uuid = Generators.timeBasedEpochGenerator().generate()
            return UserId(uuid.toString())
        }
    }
}

// domain/Email.kt
@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "無効なメールアドレス形式です" }
        require(value.length <= 255) { "メールアドレスは255文字以内です" }
    }
}

// domain/UserName.kt
@JvmInline
value class UserName(val value: String) {
    init {
        require(value.isNotBlank()) { "ユーザー名は必須です" }
        require(value.length in 1..50) { "ユーザー名は1〜50文字です" }
    }
}
```

### リポジトリインターフェース

フレームワーク非依存のインターフェースとして定義する。

```kotlin
// domain/UserRepository.kt
interface UserRepository {
    suspend fun save(user: User): User
    suspend fun findById(id: UserId): User?
    suspend fun findByEmail(email: Email): User?
    suspend fun existsByEmail(email: Email): Boolean
}
```

### ドメインサービス

複数エンティティにまたがるビジネスルールを配置する。

```kotlin
// domain/OrderPolicy.kt
class OrderPolicy {
    fun canCreateOrder(user: User, cart: Cart): Boolean {
        if (user.status != UserStatus.ACTIVE) return false
        if (cart.isEmpty()) return false
        if (cart.totalAmount() > user.creditLimit) return false
        return true
    }
}
```

---

## アプリケーション層

ユースケースの流れを組み立てる。外部連携やトランザクション境界を担当。

```kotlin
// application/UserService.kt
@Service
class UserService(
    private val userRepository: UserRepository,
    private val notificationGateway: NotificationGateway
) {
    @Transactional
    suspend fun createUser(command: CreateUserCommand): User {
        // Value Object生成（バリデーション実行）
        val email = Email(command.email)
        val name = UserName(command.name)

        // 重複チェック（リポジトリを使った検証はapplicationの責務）
        if (userRepository.existsByEmail(email)) {
            throw UserAlreadyExistsException(email)
        }

        // ドメインオブジェクト生成
        val user = User.create(email, name)

        // 永続化
        val savedUser = userRepository.save(user)

        // 副作用（通知など）
        notificationGateway.sendWelcomeEmail(savedUser)

        return savedUser
    }
}

// application/command/CreateUserCommand.kt
data class CreateUserCommand(
    val email: String,
    val name: String
)
```

### domain と application の責務分割

| 置き場所            | ロジック例                                                             |
| ------------------- | ---------------------------------------------------------------------- |
| Entity              | 自身の状態遷移、整合性チェック                                         |
| Value Object        | 値のバリデーション、等価性                                             |
| Domain Service      | 複数エンティティにまたがるビジネスルール                               |
| Application Service | ユースケースの流れ、外部連携、トランザクション、リポジトリを使った検証 |

---

## インフラストラクチャ層

リポジトリインターフェースの実装を提供する。

```kotlin
// infrastructure/UserRepositoryImpl.kt
@Repository
class UserRepositoryImpl(
    private val r2dbcEntityTemplate: R2dbcEntityTemplate
) : UserRepository {

    override suspend fun save(user: User): User {
        return r2dbcEntityTemplate.insert(user).awaitSingle()
    }

    override suspend fun findById(id: UserId): User? {
        return r2dbcEntityTemplate
            .select(User::class.java)
            .matching(query(where("id").`is`(id.value)))
            .one()
            .awaitSingleOrNull()
    }

    override suspend fun findByEmail(email: Email): User? {
        return r2dbcEntityTemplate
            .select(User::class.java)
            .matching(query(where("email").`is`(email.value)))
            .one()
            .awaitSingleOrNull()
    }

    override suspend fun existsByEmail(email: Email): Boolean {
        return r2dbcEntityTemplate
            .select(User::class.java)
            .matching(query(where("email").`is`(email.value)))
            .exists()
            .awaitSingle()
    }
}
```

---

## プレゼンテーション層

### WebFlux ルーティング方式

Functional Endpoints を採用する。

```kotlin
// presentation/UserRouter.kt
@Configuration
class UserRouter(private val handler: UserHandler) {

    @Bean
    fun userRoutes(): RouterFunction<ServerResponse> = coRouter {
        "/api/users".nest {
            GET("", handler::findAll)
            GET("/{id}", handler::findById)
            POST("", handler::create)
        }
    }
}
```

### Handler

HTTP リクエスト/レスポンスの変換に専念する。ビジネスロジックを持たない。

```kotlin
// presentation/UserHandler.kt
@Component
class UserHandler(private val userService: UserService) {

    suspend fun findAll(request: ServerRequest): ServerResponse {
        val users = userService.findAll()
        return ServerResponse.ok().bodyValueAndAwait(users.map { it.toResponse() })
    }

    suspend fun findById(request: ServerRequest): ServerResponse {
        val id = UserId(request.pathVariable("id"))
        val user = userService.findById(id)
            ?: return ServerResponse.notFound().buildAndAwait()
        return ServerResponse.ok().bodyValueAndAwait(user.toResponse())
    }

    suspend fun create(request: ServerRequest): ServerResponse {
        val req = request.awaitBody<CreateUserRequest>()
        val command = CreateUserCommand(email = req.email, name = req.name)
        val user = userService.createUser(command)
        return ServerResponse
            .created(URI.create("/api/users/${user.id.value}"))
            .bodyValueAndAwait(user.toResponse())
    }
}
```

### DTO

リクエスト/レスポンスを分離する。

```kotlin
// presentation/dto/UserRequest.kt
data class CreateUserRequest(
    val email: String,
    val name: String
)

// presentation/dto/UserResponse.kt
data class UserResponse(
    val id: String,
    val email: String,
    val name: String,
    val status: String
)

fun User.toResponse() = UserResponse(
    id = id.value,
    email = email.value,
    name = name.value,
    status = status.name
)
```

### Functional Endpoints を採用する理由

- ルーティング定義が一箇所に集約され、API の全体像が把握しやすい
- Handler がフレームワーク非依存に近づき、テストが容易
- 条件分岐を含む柔軟なルーティングが可能

---

## ID 戦略

UUID v7 を採用する。

### 採用理由

| 特性             | 説明                                             |
| ---------------- | ------------------------------------------------ |
| 時系列ソート可能 | タイムスタンプベースで生成順にソートできる       |
| インデックス効率 | B-tree インデックスへの挿入が常に末尾付近になる  |
| 分散環境対応     | DB に依存せずアプリケーション側で生成可能        |
| ドメイン独立     | 永続化前に ID が確定し、domain が infra から独立 |

### 実装

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.fasterxml.uuid:java-uuid-generator:5.0.0")
}

// domain/UserId.kt
@JvmInline
value class UserId(val value: String) {
    companion object {
        fun generate(): UserId {
            val uuid = Generators.timeBasedEpochGenerator().generate()
            return UserId(uuid.toString())
        }
    }
}
```

### 他の戦略との比較

| 戦略           | メリット                                | デメリット                     |
| -------------- | --------------------------------------- | ------------------------------ |
| UUID v4        | シンプル                                | ランダムでインデックス効率悪い |
| UUID v7        | 時系列ソート可、インデックス効率良好    | 外部ライブラリ必要             |
| ULID           | 時系列ソート可、文字列が短い（26 文字） | 外部ライブラリ必要             |
| Auto Increment | RDB と親和性高い                        | infra 依存、分散環境で課題     |

---

## テスト構成

```
test/
├── domain/
│   └── user/
│       └── UserTest.kt                 # 純粋な単体テスト
├── application/
│   └── user/
│       └── UserServiceTest.kt          # Repositoryをモック
└── infrastructure/
    └── user/
        └── UserRepositoryImplTest.kt   # @DataR2dbcTestで実DB接続
```

### domain 層のテスト

依存なし、純粋なロジックテスト。

```kotlin
class UserTest {
    @Test
    fun `新規作成時はPENDINGステータス`() {
        val user = User.create(Email("test@example.com"), UserName("Test"))
        assertEquals(UserStatus.PENDING, user.status)
    }

    @Test
    fun `PENDINGのユーザーは有効化できる`() {
        val user = User.create(Email("test@example.com"), UserName("Test"))
        val activated = user.activate()
        assertEquals(UserStatus.ACTIVE, activated.status)
    }
}
```

### application 層のテスト

Repository をモックして単体テスト。

```kotlin
class UserServiceTest {
    private val userRepository = mockk<UserRepository>()
    private val notificationGateway = mockk<NotificationGateway>()
    private val userService = UserService(userRepository, notificationGateway)

    @Test
    fun `メールアドレス重複時は例外`() = runTest {
        coEvery { userRepository.existsByEmail(any()) } returns true

        assertThrows<UserAlreadyExistsException> {
            userService.createUser(CreateUserCommand("existing@example.com", "Test"))
        }
    }
}
```

### infrastructure 層のテスト

実際の DB 接続を使った統合テスト。

```kotlin
@DataR2dbcTest
@Import(UserRepositoryImpl::class)
class UserRepositoryImplTest {
    @Autowired
    private lateinit var userRepository: UserRepository

    @Test
    fun `ユーザーを保存して取得できる`() = runTest {
        val user = User.create(Email("test@example.com"), UserName("Test"))
        val saved = userRepository.save(user)

        val found = userRepository.findById(saved.id)
        assertEquals(saved, found)
    }
}
```

---

## モジュール分割

初期段階ではシングルモジュールで開始し、以下の兆候が見られた場合にマルチモジュール化を検討する。

### マルチモジュール化の判断基準

- ビルド時間が著しく増加した場合
- 機能間の依存関係を強制的に制限したい場合
- 一部の機能を別サービスとして切り出す計画がある場合
- チームが分割され、コードオーナーシップを明確にしたい場合

### マルチモジュール構成例

```
my-app/
├── build.gradle.kts
├── settings.gradle.kts
├── app/                        # アプリケーションモジュール（エントリーポイント）
│   └── src/main/kotlin/
├── domain/                     # ドメインモジュール（ビジネスロジック）
│   └── src/main/kotlin/
├── infra/                      # インフラモジュール（外部連携）
│   └── src/main/kotlin/
└── common/                     # 共通モジュール
    └── src/main/kotlin/
```

---

## CQRS の適用

Command（更新系）と Query（参照系）で責務を分離することを検討する。ただし、すべての機能に適用する必要はない。

### 適用を検討するケース

- 参照と更新で異なる最適化が必要な場合
- 参照系のパフォーマンス要件が厳しい場合
- ドメインモデルとビューモデルが大きく乖離している場合

### 適用しないケース

- 単純な CRUD 操作のみの場合
- 参照と更新のモデルがほぼ同一の場合

### CQRS 適用時のパッケージ構成例

```
com.example.myapp/
├── domain/
│   └── user/
│       ├── User.kt
│       └── UserRepository.kt
├── application/
│   └── user/
│       ├── command/
│       │   └── CreateUserService.kt
│       └── query/
│           └── GetUserQueryService.kt
├── infrastructure/
│   └── user/
│       └── UserRepositoryImpl.kt
└── presentation/
    └── user/
        ├── UserRouter.kt           # Command/Query両方のルートを定義
        ├── command/
        │   └── CreateUserHandler.kt
        └── query/
            ├── GetUserHandler.kt
            └── dto/
                └── UserResponse.kt
```

---

## 命名規則

| 種別       | 命名規則         | 例                            |
| ---------- | ---------------- | ----------------------------- |
| パッケージ | 小文字、単数形   | `user`, `order`, `config`     |
| クラス     | PascalCase       | `UserService`, `OrderHandler` |
| ファイル   | クラス名と一致   | `UserService.kt`              |
| 定数       | UPPER_SNAKE_CASE | `MAX_RETRY_COUNT`             |
| 関数・変数 | camelCase        | `findById`, `userName`        |
