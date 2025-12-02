# 依存性注入

## 基本方針

**コンストラクタインジェクションを使用する。フィールドインジェクションは使用しない。**

## コンストラクタインジェクション

### 推奨パターン

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository,
    private val emailService: EmailService,
    private val clock: Clock,
) {
    suspend fun createUser(request: CreateUserRequest): User {
        // IDはAuto Incrementでnull（DB採番前）
        val user = User(
            id = null,
            name = request.name,
            email = request.email,
            createdAt = clock.instant(),
        )
        val savedUser = userRepository.save(user)
        emailService.sendWelcomeEmail(savedUser)
        return savedUser
    }
}
```

Kotlin ではプライマリコンストラクタで依存性を宣言するだけでよい。`@Autowired`アノテーションは不要。

### 非推奨: フィールドインジェクション

```kotlin
// 非推奨
@Service
class UserService {
    @Autowired
    private lateinit var userRepository: UserRepository

    @Autowired
    private lateinit var emailService: EmailService
}
```

### フィールドインジェクションを避ける理由

1. **テストが困難**: モックの注入にリフレクションが必要
2. **不変性の欠如**: `lateinit var`は再代入可能で、状態が変わりうる
3. **依存関係の隠蔽**: クラスのシグネチャから依存関係が見えない
4. **循環依存の検出が遅れる**: 実行時まで問題が発覚しない
5. **NullPointerException のリスク**: 初期化前にアクセスすると例外

### コンストラクタインジェクションの利点

1. **テストが容易**: コンストラクタ経由でモックを渡せる
2. **不変性**: `val`で宣言でき、再代入不可
3. **依存関係が明示的**: クラス定義を見れば依存関係がわかる
4. **循環依存の早期検出**: アプリケーション起動時に検出
5. **必須依存関係の保証**: インスタンス生成時に全依存が揃う

## テスタビリティを考慮した設計

### テストしやすいクラス設計

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val paymentGateway: PaymentGateway,
    private val tokenGenerator: TokenGenerator,  // ランダムトークン生成（副作用）
    private val clock: Clock,                    // 時刻のテスト容易性
) {
    suspend fun createOrder(request: CreateOrderRequest): Order {
        // ランダムトークンを生成（通知用など）
        val notificationToken = tokenGenerator.generateToken()

        // IDはAuto Incrementでnull（DB採番前）
        val order = Order(
            id = null,
            items = request.items,
            notificationToken = notificationToken,
            createdAt = clock.instant(),
        )
        return orderRepository.save(order)
    }
}
```

### テストコード

```kotlin
class OrderServiceTest {
    private val orderRepository = mockk<OrderRepository>()
    private val paymentGateway = mockk<PaymentGateway>()
    private val tokenGenerator = mockk<TokenGenerator>()
    private val fixedClock = Clock.fixed(
        Instant.parse("2024-01-15T10:00:00Z"),
        ZoneOffset.UTC
    )

    private val orderService = OrderService(
        orderRepository = orderRepository,
        paymentGateway = paymentGateway,
        tokenGenerator = tokenGenerator,
        clock = fixedClock,
    )

    @Test
    fun `createOrder should create order with generated token and current time`() = runTest {
        // Given
        val expectedToken = "test-token-123"
        every { tokenGenerator.generateToken() } returns expectedToken
        coEvery { orderRepository.save(any()) } returnsArgument 0

        // When
        val order = orderService.createOrder(CreateOrderRequest(...))

        // Then
        assertThat(order.notificationToken).isEqualTo(expectedToken)
        assertThat(order.createdAt).isEqualTo(fixedClock.instant())
    }
}
```

### テスト困難な設計を避ける

```kotlin
// 非推奨: 直接インスタンス化
@Service
class OrderService(
    private val orderRepository: OrderRepository,
) {
    suspend fun createOrder(request: CreateOrderRequest): Order {
        val order = Order(
            id = null,
            createdAt = Instant.now(),                   // テストで制御不可
            notificationToken = UUID.randomUUID().toString(),  // テストで制御不可
            ...
        )
        return orderRepository.save(order)
    }
}

// 非推奨: staticメソッドの直接呼び出し
@Service
class UserService {
    fun validateEmail(email: String): Boolean {
        return EmailValidator.isValid(email)  // モック不可
    }
}
```

## インターフェースの活用

### インターフェースを定義するケース

外部システムとの連携や、複数の実装が想定される場合はインターフェースを定義する。

```kotlin
// インターフェース定義
interface PaymentGateway {
    suspend fun charge(amount: Money, paymentMethod: PaymentMethod): PaymentResult
    suspend fun refund(paymentId: PaymentId, amount: Money): RefundResult
}

// 本番実装
@Component
class StripePaymentGateway(
    private val stripeClient: StripeClient,
) : PaymentGateway {
    override suspend fun charge(amount: Money, paymentMethod: PaymentMethod): PaymentResult {
        // Stripe APIを呼び出す
    }

    override suspend fun refund(paymentId: PaymentId, amount: Money): RefundResult {
        // Stripe APIを呼び出す
    }
}

// テスト用のFake実装
class FakePaymentGateway : PaymentGateway {
    private val charges = mutableListOf<Charge>()

    override suspend fun charge(amount: Money, paymentMethod: PaymentMethod): PaymentResult {
        val charge = Charge(generateId(), amount, paymentMethod)
        charges.add(charge)
        return PaymentResult.Success(charge.id)
    }

    override suspend fun refund(paymentId: PaymentId, amount: Money): RefundResult {
        // テスト用の実装
    }

    // テストヘルパー
    fun getCharges(): List<Charge> = charges.toList()
    fun clear() = charges.clear()
}
```

### インターフェースを定義しないケース

アプリケーション内部のロジックで、実装が 1 つしか想定されない場合は具象クラスを直接使用する。

```kotlin
// インターフェース不要: 実装が1つのみ
@Service
class UserService(
    private val userRepository: UserRepository,
    private val emailService: EmailService,  // 具象クラスを直接注入
) {
    // ...
}

@Service
class EmailService(
    private val emailClient: EmailClient,
    private val templateEngine: TemplateEngine,
) {
    suspend fun sendWelcomeEmail(user: User) {
        // ...
    }
}
```

**判断基準:**

- 外部システム連携 → インターフェース定義
- テストで Fake 実装を使いたい → インターフェース定義
- 複数の実装が想定される → インターフェース定義
- 上記以外 → 具象クラスを直接使用

## 設定値の注入

### @ConfigurationProperties の使用

```kotlin
@ConfigurationProperties(prefix = "app.payment")
data class PaymentProperties(
    val apiKey: String,
    val apiSecret: String,
    val timeout: Duration = Duration.ofSeconds(30),
    val retryCount: Int = 3,
)

// application.yml
app:
  payment:
    api-key: ${PAYMENT_API_KEY}
    api-secret: ${PAYMENT_API_SECRET}
    timeout: 60s
    retry-count: 5
```

### 設定クラスの注入

```kotlin
@Service
class PaymentService(
    private val paymentProperties: PaymentProperties,
    private val httpClient: HttpClient,
) {
    suspend fun processPayment(request: PaymentRequest): PaymentResult {
        val response = httpClient.post(endpoint) {
            timeout(paymentProperties.timeout)
            header("Authorization", "Bearer ${paymentProperties.apiKey}")
            // ...
        }
        // ...
    }
}
```

### @Value を避ける

```kotlin
// 非推奨: @Valueによる個別注入
@Service
class PaymentService(
    @Value("\${app.payment.api-key}") private val apiKey: String,
    @Value("\${app.payment.api-secret}") private val apiSecret: String,
    @Value("\${app.payment.timeout:30s}") private val timeout: Duration,
) {
    // ...
}
```

**@Value を避ける理由:**

- 設定値が散在し、一覧性が低い
- 型安全性が低い（文字列ベース）
- バリデーションが困難
- テスト時のモックが煩雑

## 循環依存の回避

### 循環依存の例

```kotlin
// 循環依存: 起動時にエラー
@Service
class ServiceA(private val serviceB: ServiceB) { ... }

@Service
class ServiceB(private val serviceA: ServiceA) { ... }
```

### 解決策 1: 責務の見直し

循環依存は設計の問題を示唆していることが多い。まず責務の分割を検討する。

```kotlin
// 共通ロジックを別クラスに抽出
@Service
class ServiceA(private val commonService: CommonService) { ... }

@Service
class ServiceB(private val commonService: CommonService) { ... }

@Service
class CommonService { ... }
```

### 解決策 2: イベント駆動

直接の依存関係をイベントで疎結合にする。

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val applicationEventPublisher: ApplicationEventPublisher,
) {
    suspend fun completeOrder(orderId: OrderId) {
        val order = orderRepository.findById(orderId)
        order.complete()
        orderRepository.save(order)
        applicationEventPublisher.publishEvent(OrderCompletedEvent(orderId))
    }
}

@Service
class InventoryService(
    private val inventoryRepository: InventoryRepository,
) {
    @EventListener
    suspend fun onOrderCompleted(event: OrderCompletedEvent) {
        // 在庫を更新
    }
}
```

### 解決策 3: 遅延注入（最終手段）

どうしても循環依存を解消できない場合の最終手段。

```kotlin
@Service
class ServiceA(
    private val serviceBProvider: ObjectProvider<ServiceB>,
) {
    private val serviceB: ServiceB
        get() = serviceBProvider.getObject()
}
```

## スコープ

### デフォルトスコープ（Singleton）

特に指定がなければ Singleton スコープとなる。ほとんどのサービスは Singleton で問題ない。

```kotlin
@Service  // Singleton
class UserService(...)
```

### Prototype スコープ

リクエストごとに新しいインスタンスが必要な場合に使用。ただし、WebFlux では使用場面が限られる。

```kotlin
@Component
@Scope("prototype")
class RequestScopedProcessor { ... }
```

### 注意: Singleton に Prototype を注入する問題

Singleton に Prototype を直接注入すると、Prototype のインスタンスが 1 つだけ生成されてしまう。

```kotlin
// 問題: prototypeProcessorは1回だけ生成される
@Service
class MyService(
    private val prototypeProcessor: PrototypeProcessor,  // 常に同じインスタンス
)

// 解決: ObjectProviderを使用
@Service
class MyService(
    private val processorProvider: ObjectProvider<PrototypeProcessor>,
) {
    fun process() {
        val processor = processorProvider.getObject()  // 毎回新しいインスタンス
        // ...
    }
}
```
