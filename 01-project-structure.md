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
├── shared/                             # 共有コンポーネント
│   ├── exception/
│   │   ├── ApplicationException.kt
│   │   └── GlobalExceptionHandler.kt
│   ├── extension/
│   │   └── ...
│   └── util/
│       ├── Clock.kt
│       ├── TokenGenerator.kt
│       └── ...
├── domain/                             # ドメイン層
│   ├── user/
│   │   ├── User.kt                     # エンティティ（@Table付き）
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
│   ├── order/
│   │   └── OrderRepositoryImpl.kt
│   └── external/
│       ├── payment/
│       │   └── PaymentGatewayAdapter.kt
│       └── notification/
│           └── EmailServiceAdapter.kt
└── presentation/                       # プレゼンテーション層
    ├── user/
    │   ├── UserController.kt
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
| presentation   | HTTP 層の処理        | Controller、DTO                                             |

---

## ドメイン層

### エンティティ

エンティティは DB マッピングアノテーションを持つが、ビジネスロジックは純粋に保つ。

```kotlin
// domain/User.kt
@Table("users")
data class User private constructor(
    @Id val id: Long?,  // 新規作成時はnull、DB採番後に設定
    val email: String,
    val name: String,
    val status: UserStatus,
    val createdAt: Instant
) {
    companion object {
        fun create(email: String, name: String): User {
            return User(
                id = null,
                email = email,
                name = name,
                status = UserStatus.PENDING,
                createdAt = Instant.now()
            )
        }
    }

    // 状態遷移メソッド
    fun activate(): User {
        check(status == UserStatus.PENDING) { "PENDINGのユーザーのみ有効化できます" }
        return copy(status = UserStatus.ACTIVE)
    }
}
```

### Value Object

基本的にはプリミティブ型を使用する。**バリデーションは Presentation 層の Request DTO で完全に行う前提**とする。

Value Object は以下の条件を**すべて**満たす場合のみ採用を検討する：

- 複雑なドメインルールがあり、型で表現する価値が高い
- R2DBC や Jackson のカスタムコンバーターを実装するコストを許容できる
- 型の混同による重大なバグのリスクがある

多くの場合、プリミティブ型で十分である。

### バリデーション戦略

```kotlin
// presentation/dto/UserRequest.kt
data class CreateUserRequest(
    @field:Email
    @field:NotBlank
    @field:Size(max = 255)
    val email: String,

    @field:NotBlank
    @field:Size(min = 1, max = 50)
    val name: String
)

// application/UserService.kt
@Service
class UserService(private val userRepository: UserRepository) {

    suspend fun createUser(command: CreateUserCommand): User {
        // Request DTOで完全にバリデーション済み
        // ドメインオブジェクト生成
        val user = User.create(
            email = command.email,
            name = command.name
        )
        return userRepository.save(user)
    }
}
```

### エラーハンドリング

Application/Domain 層で IllegalArgumentException が発生するのは実装バグとして扱う。

```kotlin
// common/exception/GlobalExceptionHandler.kt
@ControllerAdvice
class GlobalExceptionHandler {

    // Presentation層でのバリデーションエラー（クライアントの入力ミス）
    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidationError(e: MethodArgumentNotValidException): ResponseEntity<ErrorResponse> {
        return ResponseEntity
            .badRequest()
            .body(ErrorResponse(message = "Invalid input"))
    }

    // Application/Domain層でのバリデーションエラー（実装バグ）
    @ExceptionHandler(IllegalArgumentException::class)
    fun handleIllegalArgument(e: IllegalArgumentException): ResponseEntity<ErrorResponse> {
        logger.error("Domain validation error - this should not happen in production", e)
        return ResponseEntity
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse(message = "Internal server error"))
    }
}
```

### リポジトリインターフェース

フレームワーク非依存のインターフェースとして定義する。

```kotlin
// domain/UserRepository.kt
interface UserRepository {
    suspend fun save(user: User): User
    suspend fun findById(id: Long): User?
    suspend fun findByEmail(email: String): User?
    suspend fun existsByEmail(email: String): Boolean
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
        // 重複チェック（リポジトリを使った検証はapplicationの責務）
        if (userRepository.existsByEmail(command.email)) {
            throw UserAlreadyExistsException(command.email)
        }

        // ドメインオブジェクト生成（Request DTOで完全にバリデーション済み）
        val user = User.create(
            email = command.email,
            name = command.name
        )

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
| Value Object        | 値のバリデーション、変換ロジック、等価性                               |
| Domain Service      | 複数エンティティにまたがるビジネスルール                               |
| Application Service | ユースケースの流れ、外部連携、トランザクション、リポジトリを使った検証 |

---

## インフラストラクチャ層

### リポジトリ実装

リポジトリインターフェースの実装を提供する。

```kotlin
// infrastructure/user/UserRepositoryImpl.kt
@Repository
class UserRepositoryImpl(
    private val r2dbcEntityTemplate: R2dbcEntityTemplate
) : UserRepository {

    override suspend fun save(user: User): User {
        return r2dbcEntityTemplate.insert(user).awaitSingle()
    }

    override suspend fun findById(id: Long): User? {
        return r2dbcEntityTemplate
            .select(User::class.java)
            .matching(query(where("id").`is`(id)))
            .one()
            .awaitSingleOrNull()
    }

    override suspend fun findByEmail(email: String): User? {
        return r2dbcEntityTemplate
            .select(User::class.java)
            .matching(query(where("email").`is`(email)))
            .one()
            .awaitSingleOrNull()
    }

    override suspend fun existsByEmail(email: String): Boolean {
        return r2dbcEntityTemplate
            .select(User::class.java)
            .matching(query(where("email").`is`(email)))
            .exists()
            .awaitSingle()
    }
}
```

### 外部 API 連携（Port/Adapter パターン）

外部サービスとの連携には**Port/Adapter パターン**（Hexagonal Architecture）を採用する。

#### Domain 層で Port Interface を定義

```kotlin
// domain/payment/PaymentGateway.kt
interface PaymentGateway {
    suspend fun charge(amount: Money, token: String): PaymentResult
    suspend fun refund(paymentId: String, amount: Money): RefundResult
}

data class PaymentResult(
    val paymentId: String,
    val status: PaymentStatus,
    val transactionId: String,
)

enum class PaymentStatus {
    SUCCESS, FAILED, PENDING
}

data class RefundResult(
    val refundId: String,
    val status: RefundStatus,
)

enum class RefundStatus {
    SUCCESS, FAILED
}
```

```kotlin
// domain/notification/EmailService.kt
interface EmailService {
    suspend fun sendWelcomeEmail(email: String, name: String)
    suspend fun sendOrderConfirmation(email: String, orderId: String)
}
```

#### 設定クラスの定義

```kotlin
// config/ExternalApiProperties.kt
@ConfigurationProperties(prefix = "app.external")
data class ExternalApiProperties(
    val payment: PaymentApiConfig,
    val email: EmailApiConfig,
)

data class PaymentApiConfig(
    val baseUrl: String,
    val apiKey: String,
    val timeout: Duration = Duration.ofSeconds(30),
)

data class EmailApiConfig(
    val baseUrl: String,
    val apiKey: String,
    val timeout: Duration = Duration.ofSeconds(10),
)
```

#### Infrastructure 層で Adapter を実装

AdapterはWebClientを内部で生成し、設定も一緒に管理する。

```kotlin
// infrastructure/external/payment/PaymentGatewayAdapter.kt
@Component
class PaymentGatewayAdapter(
    externalApiProperties: ExternalApiProperties,
) : PaymentGateway {

    private val config = externalApiProperties.payment
    private val webClient: WebClient = WebClient.builder()
        .baseUrl(config.baseUrl)
        .clientConnector(
            ReactorClientHttpConnector(
                HttpClient.create()
                    .responseTimeout(config.timeout)
                    .doOnConnected { connection ->
                        connection
                            .addHandlerLast(ReadTimeoutHandler(config.timeout.seconds, TimeUnit.SECONDS))
                            .addHandlerLast(WriteTimeoutHandler(config.timeout.seconds, TimeUnit.SECONDS))
                    }
            )
        )
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .build()

    override suspend fun charge(amount: Money, token: String): PaymentResult {
        return webClient.post()
            .uri("/charges")
            .header("Authorization", "Bearer ${config.apiKey}")
            .bodyValue(
                mapOf(
                    "amount" to amount.value,
                    "currency" to "JPY",
                    "source" to token
                )
            )
            .retrieve()
            .awaitBody<PaymentApiResponse>()
            .toPaymentResult()
    }

    override suspend fun refund(paymentId: String, amount: Money): RefundResult {
        return webClient.post()
            .uri("/refunds")
            .header("Authorization", "Bearer ${config.apiKey}")
            .bodyValue(
                mapOf(
                    "charge" to paymentId,
                    "amount" to amount.value
                )
            )
            .retrieve()
            .awaitBody<RefundApiResponse>()
            .toRefundResult()
    }

    // 外部APIのレスポンス型
    private data class PaymentApiResponse(
        val id: String,
        val status: String,
        val transactionId: String,
    )

    private fun PaymentApiResponse.toPaymentResult() = PaymentResult(
        paymentId = id,
        status = when (status) {
            "succeeded" -> PaymentStatus.SUCCESS
            "failed" -> PaymentStatus.FAILED
            else -> PaymentStatus.PENDING
        },
        transactionId = transactionId,
    )
}
```

```kotlin
// infrastructure/external/notification/EmailServiceAdapter.kt
@Component
class EmailServiceAdapter(
    externalApiProperties: ExternalApiProperties,
) : EmailService {

    private val config = externalApiProperties.email
    private val webClient: WebClient = WebClient.builder()
        .baseUrl(config.baseUrl)
        .clientConnector(
            ReactorClientHttpConnector(
                HttpClient.create()
                    .responseTimeout(config.timeout)
            )
        )
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .build()

    override suspend fun sendWelcomeEmail(email: String, name: String) {
        webClient.post()
            .uri("/send")
            .header("Authorization", "Bearer ${config.apiKey}")
            .bodyValue(
                mapOf(
                    "to" to email,
                    "template" to "welcome",
                    "variables" to mapOf("name" to name)
                )
            )
            .retrieve()
            .awaitBodilessEntity()
    }

    override suspend fun sendOrderConfirmation(email: String, orderId: String) {
        webClient.post()
            .uri("/send")
            .header("Authorization", "Bearer ${config.apiKey}")
            .bodyValue(
                mapOf(
                    "to" to email,
                    "template" to "order-confirmation",
                    "variables" to mapOf("orderId" to orderId)
                )
            )
            .retrieve()
            .awaitBodilessEntity()
    }
}
```

#### application.yml設定例

```yaml
app:
  external:
    payment:
      baseUrl: ${APP_EXTERNAL_PAYMENT_BASEURL}
      apiKey: ${APP_EXTERNAL_PAYMENT_APIKEY}
      timeout: 30s
    email:
      baseUrl: ${APP_EXTERNAL_EMAIL_BASEURL}
      apiKey: ${APP_EXTERNAL_EMAIL_APIKEY}
      timeout: 10s
```

#### Application 層での使用

```kotlin
// application/order/OrderService.kt
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val paymentGateway: PaymentGateway,  // Portインターフェース
    private val emailService: EmailService,      // Portインターフェース
) {
    @Transactional
    suspend fun createOrder(command: CreateOrderCommand): Order {
        // 決済処理
        val paymentResult = paymentGateway.charge(
            amount = command.totalAmount,
            token = command.paymentToken
        )

        if (paymentResult.status != PaymentStatus.SUCCESS) {
            throw PaymentFailedException("Payment failed")
        }

        // 注文作成
        val order = Order.create(
            userId = command.userId,
            items = command.items,
            paymentId = paymentResult.paymentId
        )

        val savedOrder = orderRepository.save(order)

        // 確認メール送信（非同期・失敗しても注文は成功扱い）
        try {
            emailService.sendOrderConfirmation(
                email = command.email,
                orderId = savedOrder.id.toString()
            )
        } catch (e: Exception) {
            logger.warn("Failed to send order confirmation email", e)
        }

        return savedOrder
    }
}
```

### Port/Adapter パターンの利点

1. **テスト容易性**: Domain インターフェースをモックして単体テスト可能
2. **疎結合**: 外部サービスの実装詳細が Domain/Application 層に漏れない
3. **置き換え容易**: 外部サービスを変更してもインターフェースはそのまま
4. **明確な境界**: Domain 層が外部システムに依存しない

---

## プレゼンテーション層

### アノテーションベースのコントローラー

Spring WebFlux の`@RestController`を使用したアノテーションベースのコントローラーを採用する。

```kotlin
// presentation/UserController.kt
@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {

    @GetMapping
    suspend fun findAll(): List<UserResponse> {
        val users = userService.findAll()
        return users.map { it.toResponse() }
    }

    @GetMapping("/{id}")
    suspend fun findById(@PathVariable id: Long): UserResponse {
        val user = userService.findById(id)
            ?: throw UserNotFoundException(id)
        return user.toResponse()
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun create(@Valid @RequestBody request: CreateUserRequest): UserResponse {
        val command = CreateUserCommand(email = request.email, name = request.name)
        val user = userService.createUser(command)
        return user.toResponse()
    }

    @PutMapping("/{id}")
    suspend fun update(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateUserRequest
    ): UserResponse {
        val user = userService.update(id, request)
        return user.toResponse()
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    suspend fun delete(@PathVariable id: Long) {
        userService.delete(id)
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
    val id: Long,
    val email: String,
    val name: String,
    val status: String
)

fun User.toResponse() = UserResponse(
    id = id ?: throw IllegalStateException("IDが未設定です"),
    email = email.value,
    name = name.value,
    status = status.name
)
```

### アノテーションベースのコントローラーを採用する理由

- Spring MVC の標準的なアプローチで、多くの開発者に馴染みがある
- アノテーションによるルーティング定義が直感的
- `@Valid`によるバリデーション、`@PathVariable`、`@RequestBody`などの宣言的な記述
- OpenAPI/Swagger との統合が容易
- Spring Security のメソッドセキュリティ（`@PreAuthorize`等）と相性が良い

---

## ID 戦略

Auto Increment を採用する。

### 採用理由

| 特性             | 説明                                                   |
| ---------------- | ------------------------------------------------------ |
| RDB との親和性   | データベースネイティブの機能で高速かつ安定             |
| シーケンシャル   | 連番で生成されるためインデックス効率が非常に高い       |
| ストレージ効率   | UUID と比較して格納サイズが小さい（BIGINT: 8 バイト）  |
| デバッグの容易性 | 数値による順序が直感的で、ログやデバッグ時に扱いやすい |
| パフォーマンス   | データベースが最適化された ID 生成メカニズムを提供     |

### 実装

```kotlin
// domain/User.kt
@Table("users")
data class User private constructor(
    @Id val id: Long?,  // 新規作成時はnull、DB採番後に設定
    val email: Email,
    val name: UserName,
    val status: UserStatus,
    val createdAt: Instant
) {
    companion object {
        fun create(email: Email, name: UserName): User {
            return User(
                id = null,
                email = email,
                name = name,
                status = UserStatus.PENDING,
                createdAt = Instant.now()
            )
        }
    }
}
```

### データベーススキーマ

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(50) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

### 注意点

- **分散環境での ID 競合**: 単一データベースインスタンスでの使用を前提とする。複数のデータベースシャードを使う場合は適さない
- **事前の ID 確定不可**: 永続化前に ID が確定しないため、domain 層で ID を必要とする処理がある場合は設計を調整する必要がある
- **外部システム連携**: 外部 API との連携時に ID を事前に知る必要がある場合は、別途一意識別子（UUID など）を追加で持つことを検討

### 他の戦略との比較

| 戦略           | メリット                                | デメリット                                 |
| -------------- | --------------------------------------- | ------------------------------------------ |
| Auto Increment | RDB と親和性高い、効率的、シンプル      | infra 依存、分散環境で課題、事前 ID 不確定 |
| UUID v4        | 分散環境対応、事前 ID 確定可能          | ランダムでインデックス効率悪い             |
| UUID v7        | 時系列ソート可、インデックス効率良好    | 外部ライブラリ必要、ストレージサイズ大     |
| ULID           | 時系列ソート可、文字列が短い（26 文字） | 外部ライブラリ必要                         |

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
        val user = User.create(email = "test@example.com", name = "Test")
        assertEquals(UserStatus.PENDING, user.status)
    }

    @Test
    fun `PENDINGのユーザーは有効化できる`() {
        val user = User.create(email = "test@example.com", name = "Test")
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
        val user = User.create(email = "test@example.com", name = "Test")
        val saved = userRepository.save(user)

        val found = userRepository.findById(saved.id!!)
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
        ├── command/
        │   └── UserCommandController.kt
        └── query/
            ├── UserQueryController.kt
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
