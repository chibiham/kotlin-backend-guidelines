# テスト

## 基本方針

1. **MockK**をモッキングライブラリとして使用する
2. テストの種類に応じた適切な粒度でテストを書く
3. テストは**ドキュメント**として機能するよう、意図が明確な命名を行う
4. **Arrange-Act-Assert**（Given-When-Then）パターンに従う

## テストの種類

| 種類       | 対象                 | 実行速度 | 依存関係              |
| ---------- | -------------------- | -------- | --------------------- |
| 単体テスト | Service, Domain      | 高速     | モック                |
| 統合テスト | Repository, 外部連携 | 中速     | 実 DB, テストコンテナ |
| E2E テスト | API 全体             | 低速     | 実環境に近い構成      |

### テストピラミッド

```
        /\
       /  \     E2E（少数）
      /────\
     /      \   統合テスト（中程度）
    /────────\
   /          \ 単体テスト（多数）
  /────────────\
```

## 依存関係

```kotlin
// build.gradle.kts
dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.projectreactor:reactor-test")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test")

    // MockK
    testImplementation("io.mockk:mockk:1.13.9")
    testImplementation("com.ninja-squad:springmockk:4.0.2")

    // AssertJ
    testImplementation("org.assertj:assertj-core:3.25.1")

    // Testcontainers
    testImplementation("org.testcontainers:testcontainers:1.19.3")
    testImplementation("org.testcontainers:postgresql:1.19.3")
    testImplementation("org.testcontainers:r2dbc:1.19.3")
}
```

## MockK

### 基本的な使い方

```kotlin
class UserServiceTest {

    // モックの作成
    private val userRepository = mockk<UserRepository>()
    private val emailService = mockk<EmailService>()
    private val clock = Clock.fixed(Instant.parse("2024-01-15T10:00:00Z"), ZoneOffset.UTC)

    private val userService = UserService(
        userRepository = userRepository,
        emailService = emailService,
        clock = clock,
    )

    @BeforeEach
    fun setUp() {
        clearAllMocks()
    }

    @Test
    fun `findById should return user when exists`() = runTest {
        // Given (Arrange)
        val userId = UserId(1)
        val entity = UserEntity(id = 1, name = "John", email = "john@example.com")
        coEvery { userRepository.findById(1) } returns entity

        // When (Act)
        val result = userService.findById(userId)

        // Then (Assert)
        assertThat(result).isNotNull
        assertThat(result?.name?.value).isEqualTo("John")
        coVerify(exactly = 1) { userRepository.findById(1) }
    }

    @Test
    fun `findById should return null when not exists`() = runTest {
        // Given
        val userId = UserId(999)
        coEvery { userRepository.findById(999) } returns null

        // When
        val result = userService.findById(userId)

        // Then
        assertThat(result).isNull()
    }
}
```

### coEvery vs every

| 関数       | 用途                 |
| ---------- | -------------------- |
| `every`    | 通常の関数のモック   |
| `coEvery`  | suspend 関数のモック |
| `verify`   | 通常の関数の検証     |
| `coVerify` | suspend 関数の検証   |

```kotlin
// suspend関数のモック
coEvery { userRepository.findById(any()) } returns userEntity

// 通常の関数のモック
every { validator.validate(any()) } returns ValidationResult.Success

// suspend関数の検証
coVerify { userRepository.save(any()) }

// 通常の関数の検証
verify { logger.info(any()) }
```

### モックの振る舞い定義

```kotlin
// 固定値を返す
coEvery { repository.findById(1) } returns entity

// nullを返す
coEvery { repository.findById(999) } returns null

// 例外をスロー
coEvery { repository.findById(-1) } throws IllegalArgumentException("Invalid ID")

// 引数をそのまま返す
coEvery { repository.save(any()) } returnsArgument 0

// 引数に基づいて動的に返す
coEvery { repository.findById(any()) } answers {
    val id = firstArg<Long>()
    if (id > 0) UserEntity(id = id, name = "User$id", email = "user$id@example.com")
    else null
}

// 順番に異なる値を返す
coEvery { repository.findById(1) } returnsMany listOf(entity1, entity2, entity3)

// 最初は例外、次は成功
coEvery { externalService.call() } throws IOException() andThen response
```

### 引数のマッチング

```kotlin
// 任意の値
coEvery { repository.findById(any()) } returns entity

// 特定の値
coEvery { repository.findById(eq(1)) } returns entity

// 条件に一致
coEvery { repository.findByEmail(match { it.contains("@") }) } returns entity

// キャプチャ
val slot = slot<UserEntity>()
coEvery { repository.save(capture(slot)) } returns mockk()

// 後で検証
assertThat(slot.captured.name).isEqualTo("John")
```

### Relaxed Mock

```kotlin
// 全メソッドがデフォルト値を返すモック
val relaxedMock = mockk<UserRepository>(relaxed = true)

// Unit を返すメソッドのみ relaxed
val unitRelaxedMock = mockk<UserRepository>(relaxUnitFun = true)
```

## 単体テスト

### Service のテスト

```kotlin
class OrderServiceTest {

    private val orderRepository = mockk<OrderRepository>()
    private val inventoryService = mockk<InventoryService>()
    private val paymentGateway = mockk<PaymentGateway>()
    private val tokenGenerator = mockk<TokenGenerator>()
    private val clock = Clock.fixed(Instant.parse("2024-01-15T10:00:00Z"), ZoneOffset.UTC)

    private val orderService = OrderService(
        orderRepository = orderRepository,
        inventoryService = inventoryService,
        paymentGateway = paymentGateway,
        tokenGenerator = tokenGenerator,
        clock = clock,
    )

    @Nested
    inner class CreateOrder {

        @Test
        fun `should create order when inventory is available`() = runTest {
            // Given
            val request = CreateOrderRequest(
                userId = UserId(1),
                items = listOf(OrderItem(ProductId(100), quantity = 2)),
            )
            val savedOrder = Order(
                id = 123L,  // DB採番後のID
                userId = UserId(1),
                items = listOf(OrderItem(ProductId(100), quantity = 2)),
                status = OrderStatus.PENDING,
                createdAt = clock.instant(),
            )

            every { tokenGenerator.generateToken() } returns "test-token-123"
            coEvery { inventoryService.checkAvailability(any()) } returns true
            coEvery { inventoryService.reserve(any()) } just Runs
            coEvery { orderRepository.save(any()) } returns savedOrder

            // When
            val result = orderService.create(request)

            // Then
            assertThat(result.id).isEqualTo(123L)
            assertThat(result.status).isEqualTo(OrderStatus.PENDING)
            assertThat(result.createdAt).isEqualTo(clock.instant())
            assertThat(result.notificationToken).isEqualTo("test-token-123")

            coVerify(exactly = 1) { inventoryService.reserve(any()) }
            coVerify(exactly = 1) { orderRepository.save(any()) }
        }

        @Test
        fun `should throw exception when inventory is not available`() = runTest {
            // Given
            val request = CreateOrderRequest(
                userId = UserId(1),
                items = listOf(OrderItem(ProductId(100), quantity = 100)),
            )

            coEvery { inventoryService.checkAvailability(any()) } returns false

            // When & Then
            assertThatThrownBy {
                runBlocking { orderService.create(request) }
            }.isInstanceOf(ApplicationException.BadRequest.InsufficientStock::class.java)

            coVerify(exactly = 0) { orderRepository.save(any()) }
        }
    }

    @Nested
    inner class CancelOrder {

        @Test
        fun `should cancel order and refund payment`() = runTest {
            // Given
            val orderId = OrderId("order-123")
            val order = Order(
                id = orderId,
                status = OrderStatus.PAID,
                paymentId = PaymentId("payment-456"),
                // ...
            )

            coEvery { orderRepository.findById(orderId) } returns order
            coEvery { paymentGateway.refund(any(), any()) } returns RefundResult.Success
            coEvery { orderRepository.save(any()) } returnsArgument 0
            coEvery { inventoryService.release(any()) } just Runs

            // When
            val result = orderService.cancel(orderId)

            // Then
            assertThat(result.status).isEqualTo(OrderStatus.CANCELLED)
            coVerify { paymentGateway.refund(PaymentId("payment-456"), any()) }
            coVerify { inventoryService.release(any()) }
        }
    }
}
```

### Domain のテスト

ドメインモデルは純粋な単体テスト（モック不要）で検証する。

```kotlin
class OrderTest {

    @Test
    fun `should calculate total amount`() {
        // Given
        val order = Order(
            id = OrderId("order-1"),
            items = listOf(
                OrderItem(ProductId(1), quantity = 2, unitPrice = Money(100)),
                OrderItem(ProductId(2), quantity = 3, unitPrice = Money(200)),
            ),
        )

        // When
        val total = order.totalAmount

        // Then
        assertThat(total).isEqualTo(Money(800))  // 2*100 + 3*200
    }

    @Test
    fun `should not allow cancellation of shipped order`() {
        // Given
        val order = Order(
            id = OrderId("order-1"),
            status = OrderStatus.SHIPPED,
        )

        // When & Then
        assertThatThrownBy { order.cancel() }
            .isInstanceOf(IllegalStateException::class.java)
            .hasMessage("Cannot cancel shipped order")
    }

    @Nested
    inner class StatusTransitions {

        @ParameterizedTest
        @CsvSource(
            "PENDING, PAID, true",
            "PENDING, CANCELLED, true",
            "PAID, SHIPPED, true",
            "PAID, CANCELLED, true",
            "SHIPPED, DELIVERED, true",
            "SHIPPED, CANCELLED, false",
            "DELIVERED, CANCELLED, false",
        )
        fun `should validate status transitions`(
            from: OrderStatus,
            to: OrderStatus,
            allowed: Boolean,
        ) {
            val order = Order(id = OrderId("1"), status = from)

            if (allowed) {
                assertThatNoException().isThrownBy { order.transitionTo(to) }
            } else {
                assertThatThrownBy { order.transitionTo(to) }
                    .isInstanceOf(IllegalStateException::class.java)
            }
        }
    }
}
```

## 統合テスト

### Repository のテスト

```kotlin
@DataR2dbcTest
@Testcontainers
class UserRepositoryTest {

    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer("postgres:15-alpine")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test")

        @DynamicPropertySource
        @JvmStatic
        fun properties(registry: DynamicPropertyRegistry) {
            registry.add("spring.r2dbc.url") {
                "r2dbc:postgresql://${postgres.host}:${postgres.firstMappedPort}/${postgres.databaseName}"
            }
            registry.add("spring.r2dbc.username", postgres::getUsername)
            registry.add("spring.r2dbc.password", postgres::getPassword)
        }
    }

    @Autowired
    private lateinit var userRepository: UserRepository

    @Autowired
    private lateinit var r2dbcEntityTemplate: R2dbcEntityTemplate

    @BeforeEach
    fun setUp() = runBlocking {
        r2dbcEntityTemplate.delete(UserEntity::class.java).all().awaitSingle()
    }

    @Test
    fun `should save and find user`() = runTest {
        // Given
        val entity = UserEntity(
            name = "John Doe",
            email = "john@example.com",
            status = UserStatus.ACTIVE,
        )

        // When
        val saved = userRepository.save(entity)
        val found = userRepository.findById(saved.id!!)

        // Then
        assertThat(found).isNotNull
        assertThat(found?.name).isEqualTo("John Doe")
        assertThat(found?.email).isEqualTo("john@example.com")
    }

    @Test
    fun `should find user by email`() = runTest {
        // Given
        val entity = UserEntity(
            name = "Jane Doe",
            email = "jane@example.com",
            status = UserStatus.ACTIVE,
        )
        userRepository.save(entity)

        // When
        val found = userRepository.findByEmail("jane@example.com")

        // Then
        assertThat(found).isNotNull
        assertThat(found?.name).isEqualTo("Jane Doe")
    }

    @Test
    fun `should return null when user not found by email`() = runTest {
        // When
        val found = userRepository.findByEmail("notfound@example.com")

        // Then
        assertThat(found).isNull()
    }
}
```

### 外部 API のテスト（WireMock）

```kotlin
@SpringBootTest
@AutoConfigureWireMock(port = 0)
class ExternalApiServiceTest {

    @Autowired
    private lateinit var externalApiService: ExternalApiService

    @Test
    fun `should fetch user from external API`() = runTest {
        // Given
        stubFor(
            get(urlEqualTo("/users/123"))
                .willReturn(
                    aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""
                            {
                                "id": "123",
                                "name": "External User",
                                "email": "external@example.com"
                            }
                        """.trimIndent())
                )
        )

        // When
        val user = externalApiService.fetchUser("123")

        // Then
        assertThat(user.id).isEqualTo("123")
        assertThat(user.name).isEqualTo("External User")
    }

    @Test
    fun `should handle external API error`() = runTest {
        // Given
        stubFor(
            get(urlEqualTo("/users/999"))
                .willReturn(
                    aResponse()
                        .withStatus(500)
                        .withBody("Internal Server Error")
                )
        )

        // When & Then
        assertThatThrownBy {
            runBlocking { externalApiService.fetchUser("999") }
        }.isInstanceOf(ApplicationException.ExternalServiceError::class.java)
    }

    @Test
    fun `should retry on temporary failure`() = runTest {
        // Given
        stubFor(
            get(urlEqualTo("/users/456"))
                .inScenario("retry")
                .whenScenarioStateIs(Scenario.STARTED)
                .willReturn(aResponse().withStatus(503))
                .willSetStateTo("second-attempt")
        )
        stubFor(
            get(urlEqualTo("/users/456"))
                .inScenario("retry")
                .whenScenarioStateIs("second-attempt")
                .willReturn(
                    aResponse()
                        .withStatus(200)
                        .withHeader("Content-Type", "application/json")
                        .withBody("""{"id": "456", "name": "User"}""")
                )
        )

        // When
        val user = externalApiService.fetchUser("456")

        // Then
        assertThat(user.id).isEqualTo("456")
        verify(2, getRequestedFor(urlEqualTo("/users/456")))
    }
}
```

## WebFlux テスト

### Controller のテスト

```kotlin
@WebFluxTest(UserController::class)
class UserControllerTest {

    @Autowired
    private lateinit var webTestClient: WebTestClient

    @MockkBean
    private lateinit var userService: UserService

    @Test
    fun `GET users should return list`() {
        // Given
        val users = listOf(
            User(id = 1, name = "Alice", email = "alice@example.com"),
            User(id = 2, name = "Bob", email = "bob@example.com"),
        )
        coEvery { userService.findAll() } returns users

        // When & Then
        webTestClient.get()
            .uri("/api/users")
            .exchange()
            .expectStatus().isOk
            .expectHeader().contentType(MediaType.APPLICATION_JSON)
            .expectBody()
            .jsonPath("$.length()").isEqualTo(2)
            .jsonPath("$[0].name").isEqualTo("Alice")
            .jsonPath("$[1].name").isEqualTo("Bob")
    }

    @Test
    fun `GET users by id should return user`() {
        // Given
        val user = User(id = 1, name = "Alice", email = "alice@example.com")
        coEvery { userService.findById(1L) } returns user

        // When & Then
        webTestClient.get()
            .uri("/api/users/1")
            .exchange()
            .expectStatus().isOk
            .expectBody()
            .jsonPath("$.id").isEqualTo(1)
            .jsonPath("$.name").isEqualTo("Alice")
    }

    @Test
    fun `GET users by id should return 404 when not found`() {
        // Given
        coEvery { userService.findById(999L) } returns null

        // When & Then
        webTestClient.get()
            .uri("/api/users/999")
            .exchange()
            .expectStatus().isNotFound
            .expectHeader().contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .expectBody()
            .jsonPath("$.type").isEqualTo("https://api.example.com/problems/user-not-found")
            .jsonPath("$.status").isEqualTo(404)
    }

    @Test
    fun `POST users should create user`() {
        // Given
        val request = CreateUserRequest(name = "Charlie", email = "charlie@example.com")
        val created = User(id = 3, name = "Charlie", email = "charlie@example.com")
        coEvery { userService.create(any()) } returns created

        // When & Then
        webTestClient.post()
            .uri("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(request)
            .exchange()
            .expectStatus().isCreated
            .expectBody()
            .jsonPath("$.id").isEqualTo(3)
            .jsonPath("$.name").isEqualTo("Charlie")
    }

    @Test
    fun `POST users should return 400 for invalid request`() {
        // When & Then
        webTestClient.post()
            .uri("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"name": "", "email": "invalid"}""")
            .exchange()
            .expectStatus().isBadRequest
            .expectBody()
            .jsonPath("$.type").isEqualTo("https://api.example.com/problems/validation-error")
            .jsonPath("$.errors").isArray
    }
}
```

### セキュリティを含むテスト（Scopeベース）

```kotlin
@WebFluxTest
@Import(SecurityConfig::class)
class SecuredEndpointTest {

    @Autowired
    private lateinit var webTestClient: WebTestClient

    @MockkBean
    private lateinit var userService: UserService

    @MockkBean
    private lateinit var opaqueTokenIntrospector: ReactiveOpaqueTokenIntrospector

    @Test
    fun `should return 401 without token`() {
        webTestClient.get()
            .uri("/api/users")
            .exchange()
            .expectStatus().isUnauthorized
            .expectHeader().valueMatches(
                HttpHeaders.WWW_AUTHENTICATE,
                """Bearer realm="api".*"""
            )
    }

    @Test
    fun `should return 200 with valid scope`() {
        // Given
        coEvery { userService.findAll() } returns emptyList()
        mockTokenIntrospection("users:read")

        // When & Then
        webTestClient.get()
            .uri("/api/users")
            .headers { it.setBearerAuth("valid-token") }
            .exchange()
            .expectStatus().isOk
    }

    @Test
    fun `should return 403 without required scope`() {
        // Given
        mockTokenIntrospection("orders:read")  // users:readが必要

        // When & Then
        webTestClient.get()
            .uri("/api/users")
            .headers { it.setBearerAuth("valid-token") }
            .exchange()
            .expectStatus().isForbidden
    }

    @Test
    fun `should return 403 for delete without delete scope`() {
        // Given
        mockTokenIntrospection("users:read users:write")  // users:deleteが必要

        // When & Then
        webTestClient.delete()
            .uri("/api/users/1")
            .headers { it.setBearerAuth("valid-token") }
            .exchange()
            .expectStatus().isForbidden
    }

    @Test
    fun `should return 204 with delete scope`() {
        // Given
        coEvery { userService.delete(1L) } just Runs
        mockTokenIntrospection("users:delete")

        // When & Then
        webTestClient.delete()
            .uri("/api/users/1")
            .headers { it.setBearerAuth("valid-token") }
            .exchange()
            .expectStatus().isNoContent
    }

    private fun mockTokenIntrospection(scope: String) {
        val attributes = mapOf(
            "active" to true,
            "scope" to scope,
            "sub" to "12345",
            "client_id" to "test-client"
        )

        val authorities = scope.split(" ").map { SimpleGrantedAuthority("SCOPE_$it") }

        every { opaqueTokenIntrospector.introspect(any()) } returns Mono.just(
            OAuth2IntrospectionAuthenticatedPrincipal(
                attributes["sub"] as String,
                attributes,
                authorities
            )
        )
    }
}
```

## テストの命名規則

### メソッド名の形式

```kotlin
// 形式: should [期待する動作] when [条件]
@Test
fun `should return user when exists`() { }

@Test
fun `should throw NotFound when user does not exist`() { }

@Test
fun `should create order and reserve inventory`() { }

// または: [メソッド名] should [動作] [条件]
@Test
fun `findById should return user when exists`() { }

@Test
fun `create should throw ValidationException when email is invalid`() { }
```

### @Nested でグループ化

```kotlin
class UserServiceTest {

    @Nested
    @DisplayName("findById")
    inner class FindById {

        @Test
        fun `should return user when exists`() { }

        @Test
        fun `should return null when not exists`() { }
    }

    @Nested
    @DisplayName("create")
    inner class Create {

        @Test
        fun `should create user with valid request`() { }

        @Test
        fun `should throw DuplicateEmail when email already exists`() { }
    }
}
```

## テストフィクスチャ

### ファクトリ関数

```kotlin
// test/kotlin/fixtures/UserFixtures.kt
object UserFixtures {

    fun createUser(
        id: UserId = UserId(1),
        name: UserName = UserName("Test User"),
        email: Email = Email("test@example.com"),
        status: UserStatus = UserStatus.ACTIVE,
        createdAt: Instant = Instant.parse("2024-01-15T10:00:00Z"),
    ): User = User(
        id = id,
        name = name,
        email = email,
        status = status,
        createdAt = createdAt,
    )

    fun createUserEntity(
        id: Long? = 1,
        name: String = "Test User",
        email: String = "test@example.com",
        status: UserStatus = UserStatus.ACTIVE,
    ): UserEntity = UserEntity(
        id = id,
        name = name,
        email = email,
        status = status,
    )
}

// 使用例
@Test
fun `test with fixture`() {
    val user = UserFixtures.createUser(name = UserName("Custom Name"))
    // ...
}
```

### Builder パターン

```kotlin
class OrderBuilder {
    private var id: OrderId = OrderId("order-1")
    private var userId: UserId = UserId(1)
    private var status: OrderStatus = OrderStatus.PENDING
    private var items: List<OrderItem> = emptyList()
    private var createdAt: Instant = Instant.now()

    fun withId(id: OrderId) = apply { this.id = id }
    fun withUserId(userId: UserId) = apply { this.userId = userId }
    fun withStatus(status: OrderStatus) = apply { this.status = status }
    fun withItems(items: List<OrderItem>) = apply { this.items = items }
    fun withCreatedAt(createdAt: Instant) = apply { this.createdAt = createdAt }

    fun build(): Order = Order(
        id = id,
        userId = userId,
        status = status,
        items = items,
        createdAt = createdAt,
    )
}

// 使用例
val order = OrderBuilder()
    .withStatus(OrderStatus.PAID)
    .withItems(listOf(orderItem1, orderItem2))
    .build()
```

## テストのベストプラクティス

### 1 つのテストで 1 つのことを検証

```kotlin
// 良い例: 1つの振る舞いを検証
@Test
fun `should save user to repository`() = runTest {
    // ...
    coVerify(exactly = 1) { userRepository.save(any()) }
}

@Test
fun `should send welcome email`() = runTest {
    // ...
    coVerify(exactly = 1) { emailService.sendWelcomeEmail(any()) }
}

// 悪い例: 複数の振る舞いを1つのテストで検証
@Test
fun `should save user and send email and log`() = runTest {
    // 複数の検証が混在 - テスト失敗時に原因特定が困難
}
```

### テストの独立性

```kotlin
class UserServiceTest {

    private lateinit var userRepository: UserRepository
    private lateinit var userService: UserService

    @BeforeEach
    fun setUp() {
        // 各テストの前に初期化
        userRepository = mockk()
        userService = UserService(userRepository)
        clearAllMocks()
    }

    // 各テストは他のテストに依存しない
}
```

### 過度なモックを避ける

```kotlin
// 悪い例: 全てをモック
@Test
fun `test with too many mocks`() {
    val user = mockk<User>()
    every { user.id } returns UserId(1)
    every { user.name } returns UserName("John")
    every { user.email } returns Email("john@example.com")
    every { user.isActive() } returns true
    // ...モックだらけで何をテストしているか不明
}

// 良い例: 実オブジェクトを使用
@Test
fun `test with real objects`() {
    val user = User(
        id = UserId(1),
        name = UserName("John"),
        email = Email("john@example.com"),
        status = UserStatus.ACTIVE,
    )
    // 実際のオブジェクトの振る舞いをテスト
}
```

### 境界値のテスト

```kotlin
@Nested
inner class PasswordValidation {

    @Test
    fun `should accept password with minimum length`() {
        val result = validator.validate("12345678")  // 最小8文字
        assertThat(result.isValid).isTrue()
    }

    @Test
    fun `should reject password below minimum length`() {
        val result = validator.validate("1234567")  // 7文字
        assertThat(result.isValid).isFalse()
    }

    @Test
    fun `should accept password with maximum length`() {
        val result = validator.validate("a".repeat(100))  // 最大100文字
        assertThat(result.isValid).isTrue()
    }

    @Test
    fun `should reject password above maximum length`() {
        val result = validator.validate("a".repeat(101))  // 101文字
        assertThat(result.isValid).isFalse()
    }
}
```

### テストカバレッジ

テストカバレッジは目標ではなく指標として扱う。

```kotlin
// build.gradle.kts
plugins {
    id("org.jetbrains.kotlinx.kover") version "0.7.5"
}

kover {
    reports {
        filters {
            excludes {
                classes("*Config", "*Configuration", "*Application")
            }
        }
        verify {
            rule {
                minBound(80)  // 最低80%のカバレッジ
            }
        }
    }
}
```

**重要:** 高いカバレッジよりも、重要なビジネスロジックがテストされていることを優先する。
