# 非同期・並行処理

## 基本方針

1. **Spring WebFlux + Kotlin Coroutines**を採用する
2. ブロッキング処理は専用の Dispatcher で実行する
3. HTTP クライアントはノンブロッキング対応のものを使用する
4. 構造化された並行性（Structured Concurrency）を意識する

## Coroutines の基礎

### suspend 関数

`suspend`キーワードは、その関数が中断可能であることを示すマーカー。実際の非同期性は Coroutine のコンテキストと Dispatcher によって決まる。

```kotlin
// suspend関数の定義
suspend fun findUserById(id: UserId): User? {
    return userRepository.findById(id)  // R2DBCの場合、内部でsuspend
}

// suspend関数の呼び出しはsuspend関数内またはCoroutineスコープ内で行う
suspend fun processUser(id: UserId) {
    val user = findUserById(id)  // OK: suspend関数内
    // ...
}
```

### CoroutineScope

```kotlin
// Spring WebFluxでは、@RestControllerのsuspend関数が自動的にCoroutineScopeを提供
@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {

    @GetMapping("/{id}")
    suspend fun findById(@PathVariable id: Long): UserResponse {
        // この中はCoroutineScope内
        val user = userService.findById(id)
            ?: throw UserNotFoundException(id)
        return user.toResponse()
    }
}
```

### Dispatcher

| Dispatcher               | 用途                       | スレッドプール                   |
| ------------------------ | -------------------------- | -------------------------------- |
| `Dispatchers.Default`    | CPU 集約型の処理           | CPU コア数に応じたスレッド       |
| `Dispatchers.IO`         | I/O 処理、ブロッキング処理 | 64 スレッド（または CPU コア数） |
| `Dispatchers.Unconfined` | 特定スレッドに縛られない   | 呼び出し元のスレッド             |

```kotlin
// Dispatcherの切り替え
suspend fun processLargeData(data: List<Item>): Result {
    return withContext(Dispatchers.Default) {
        // CPU集約型の処理
        data.map { computeExpensiveOperation(it) }
    }
}

suspend fun readLegacyFile(path: String): String {
    return withContext(Dispatchers.IO) {
        // ブロッキングI/O
        File(path).readText()
    }
}
```

## WebFlux での Coroutines

### Controller 実装

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService,
) {
    // suspend関数としてControllerメソッドを定義
    @GetMapping("/{id}")
    suspend fun findById(@PathVariable id: Long): UserResponse {
        val user = userService.findById(id)
            ?: throw UserNotFoundException(id)
        return user.toResponse()
    }

    @GetMapping
    suspend fun findAll(): List<UserResponse> {
        val users = userService.findAll()
        return users.map { it.toResponse() }
    }
}
```

### Service 実装

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository,
    private val emailService: EmailService,
) {
    suspend fun findById(id: UserId): User? {
        return userRepository.findById(id.value)?.toDomain()
    }

    suspend fun findAll(): List<User> {
        return userRepository.findAll().toList().map { it.toDomain() }
    }

    suspend fun create(request: CreateUserRequest): User {
        val entity = UserEntity(
            name = request.name,
            email = request.email,
        )
        val saved = userRepository.save(entity)

        // 非同期でメール送信（結果を待たない場合）
        coroutineScope {
            launch {
                emailService.sendWelcomeEmail(saved.email)
            }
        }

        return saved.toDomain()
    }
}
```

### Repository（R2DBC）

```kotlin
interface UserRepository : CoroutineCrudRepository<UserEntity, Long> {
    suspend fun findByEmail(email: String): UserEntity?

    fun findByStatus(status: UserStatus): Flow<UserEntity>
}
```

## Flow

### Flow の基本

`Flow`は Kotlin のコールドストリーム。データは収集（collect）されるまで流れない。

```kotlin
// Flowの生成
fun generateNumbers(): Flow<Int> = flow {
    for (i in 1..10) {
        delay(100)
        emit(i)
    }
}

// Flowの収集
suspend fun processNumbers() {
    generateNumbers()
        .filter { it % 2 == 0 }
        .map { it * 2 }
        .collect { println(it) }
}
```

### Repository での Flow

```kotlin
interface OrderRepository : CoroutineCrudRepository<OrderEntity, Long> {
    // 大量データはFlowで返す
    fun findByUserId(userId: Long): Flow<OrderEntity>

    fun findByStatus(status: OrderStatus): Flow<OrderEntity>
}

@Service
class OrderService(private val orderRepository: OrderRepository) {

    // FlowをListに変換
    suspend fun findByUserId(userId: UserId): List<Order> {
        return orderRepository.findByUserId(userId.value)
            .map { it.toDomain() }
            .toList()
    }

    // Flowのまま返す（ストリーミングレスポンス用）
    fun streamByStatus(status: OrderStatus): Flow<Order> {
        return orderRepository.findByStatus(status)
            .map { it.toDomain() }
    }
}
```

### ストリーミングレスポンス

```kotlin
@Component
class OrderHandler(private val orderService: OrderService) {

    suspend fun streamOrders(request: ServerRequest): ServerResponse {
        val status = OrderStatus.valueOf(request.queryParam("status").orElse("PENDING"))
        val orders = orderService.streamByStatus(status)
            .map { it.toResponse() }

        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_NDJSON)  // Newline Delimited JSON
            .bodyAndAwait(orders)
    }
}
```

## 並行処理

### 複数の非同期処理を並行実行

```kotlin
@Service
class DashboardService(
    private val userService: UserService,
    private val orderService: OrderService,
    private val notificationService: NotificationService,
) {
    suspend fun getDashboard(userId: UserId): Dashboard {
        // 並行して複数のデータを取得
        return coroutineScope {
            val userDeferred = async { userService.findById(userId) }
            val ordersDeferred = async { orderService.findRecentByUserId(userId) }
            val notificationsDeferred = async { notificationService.findUnread(userId) }

            Dashboard(
                user = userDeferred.await() ?: throw ApplicationException.NotFound.User(userId),
                recentOrders = ordersDeferred.await(),
                unreadNotifications = notificationsDeferred.await(),
            )
        }
    }
}
```

### async と await の注意点

```kotlin
// 良い例: coroutineScopeで囲む（構造化された並行性）
suspend fun fetchAll(): Result {
    return coroutineScope {
        val a = async { fetchA() }
        val b = async { fetchB() }
        Result(a.await(), b.await())
    }
    // coroutineScope内の全処理が完了するまで待機
    // 1つでも失敗すると他もキャンセルされる
}

// 悪い例: GlobalScopeの使用
suspend fun fetchAllBad(): Result {
    // GlobalScopeは構造化されていない
    // キャンセルが伝播しない、リークの原因になる
    val a = GlobalScope.async { fetchA() }  // NG
    val b = GlobalScope.async { fetchB() }  // NG
    return Result(a.await(), b.await())
}
```

### 並列度の制御

```kotlin
// Semaphoreで同時実行数を制限
class RateLimitedService(
    private val externalApi: ExternalApi,
) {
    private val semaphore = Semaphore(10)  // 最大10並列

    suspend fun callApi(request: ApiRequest): ApiResponse {
        return semaphore.withPermit {
            externalApi.call(request)
        }
    }
}

// Flowで並列度を制御
suspend fun processItems(items: List<Item>) {
    items.asFlow()
        .flatMapMerge(concurrency = 5) { item ->
            flow { emit(processItem(item)) }
        }
        .collect { result -> saveResult(result) }
}
```

## HTTP クライアント

### WebClient（推奨）

Spring WebFlux に含まれるノンブロッキング HTTP クライアント。

```kotlin
@Configuration
class WebClientConfig {

    @Bean
    fun webClient(): WebClient {
        return WebClient.builder()
            .baseUrl("https://api.example.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .filter(logRequest())
            .filter(logResponse())
            .build()
    }

    private fun logRequest(): ExchangeFilterFunction {
        return ExchangeFilterFunction.ofRequestProcessor { request ->
            logger.debug { "Request: ${request.method()} ${request.url()}" }
            Mono.just(request)
        }
    }

    private fun logResponse(): ExchangeFilterFunction {
        return ExchangeFilterFunction.ofResponseProcessor { response ->
            logger.debug { "Response: ${response.statusCode()}" }
            Mono.just(response)
        }
    }
}
```

### WebClient の使用

```kotlin
@Service
class ExternalApiService(
    private val webClient: WebClient,
) {
    suspend fun fetchUser(externalId: String): ExternalUser {
        return webClient.get()
            .uri("/users/{id}", externalId)
            .retrieve()
            .onStatus({ it.isError }) { response ->
                response.bodyToMono<String>().map { body ->
                    ApplicationException.ExternalServiceError.Api(
                        "External API error: ${response.statusCode()}, body: $body"
                    )
                }
            }
            .awaitBody()
    }

    suspend fun createUser(request: CreateExternalUserRequest): ExternalUser {
        return webClient.post()
            .uri("/users")
            .bodyValue(request)
            .retrieve()
            .awaitBody()
    }
}
```


## ブロッキング処理の扱い

### 専用 Dispatcher で実行

ブロッキング API を使用する場合は、`Dispatchers.IO`で実行する。

```kotlin
@Service
class LegacyIntegrationService(
    private val legacyClient: LegacyBlockingClient,  // ブロッキングAPI
) {
    suspend fun fetchData(id: String): LegacyData {
        return withContext(Dispatchers.IO) {
            // ブロッキング呼び出しをIOスレッドで実行
            legacyClient.getData(id)
        }
    }
}
```

### カスタム Dispatcher の作成

特定のリソースへのアクセスを制限したい場合。

```kotlin
@Configuration
class DispatcherConfig {

    @Bean
    @Qualifier("legacyDispatcher")
    fun legacyDispatcher(): CoroutineDispatcher {
        // レガシーシステムへの同時接続数を制限
        return Executors.newFixedThreadPool(5).asCoroutineDispatcher()
    }
}

@Service
class LegacyService(
    @Qualifier("legacyDispatcher")
    private val legacyDispatcher: CoroutineDispatcher,
    private val legacyClient: LegacyClient,
) {
    suspend fun process(request: Request): Response {
        return withContext(legacyDispatcher) {
            legacyClient.process(request)
        }
    }
}
```

## タイムアウトとリトライ

### タイムアウト

```kotlin
suspend fun fetchWithTimeout(id: String): Data {
    return withTimeout(5000) {  // 5秒でタイムアウト
        externalService.fetch(id)
    }
}

// タイムアウト時にnullを返す
suspend fun fetchWithTimeoutOrNull(id: String): Data? {
    return withTimeoutOrNull(5000) {
        externalService.fetch(id)
    }
}
```

### リトライ

```kotlin
// シンプルなリトライ
suspend fun <T> retry(
    times: Int = 3,
    initialDelay: Long = 100,
    maxDelay: Long = 1000,
    factor: Double = 2.0,
    block: suspend () -> T,
): T {
    var currentDelay = initialDelay
    repeat(times - 1) { attempt ->
        try {
            return block()
        } catch (e: Exception) {
            logger.warn { "Attempt ${attempt + 1} failed: ${e.message}" }
        }
        delay(currentDelay)
        currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelay)
    }
    return block()  // 最後の試行
}

// 使用例
suspend fun fetchWithRetry(id: String): Data {
    return retry(times = 3, initialDelay = 100) {
        externalService.fetch(id)
    }
}
```

### Flow でのリトライ

```kotlin
fun fetchDataFlow(): Flow<Data> = flow {
    emit(externalService.fetch())
}.retry(3) { cause ->
    // リトライ条件
    cause is IOException
}.catch { e ->
    // エラーハンドリング
    emit(Data.empty())
}
```

## エラーハンドリング

### Coroutine 例外ハンドリング

```kotlin
// supervisorScopeで子の失敗を分離
suspend fun processMultiple(items: List<Item>): List<Result> {
    return supervisorScope {
        items.map { item ->
            async {
                try {
                    processItem(item)
                } catch (e: Exception) {
                    logger.error(e) { "Failed to process item: ${item.id}" }
                    Result.Failure(item.id, e.message)
                }
            }
        }.awaitAll()
    }
}

// CoroutineExceptionHandler
val handler = CoroutineExceptionHandler { _, exception ->
    logger.error(exception) { "Unhandled exception in coroutine" }
}

// Supervisorジョブと組み合わせ
val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default + handler)
```

### Flow のエラーハンドリング

```kotlin
fun processFlow(): Flow<Result> = flow {
    // ...
}.catch { e ->
    logger.error(e) { "Flow error" }
    emit(Result.Error(e))
}.onCompletion { cause ->
    if (cause != null) {
        logger.warn { "Flow completed with error: ${cause.message}" }
    }
}
```

## テスト

### Coroutines のテスト

```kotlin
class UserServiceTest {

    private val userRepository = mockk<UserRepository>()
    private val userService = UserService(userRepository)

    @Test
    fun `findById should return user`() = runTest {
        // Given
        val userId = UserId(1)
        val entity = UserEntity(id = 1, name = "John", email = "john@example.com")
        coEvery { userRepository.findById(1) } returns entity

        // When
        val result = userService.findById(userId)

        // Then
        assertThat(result?.name?.value).isEqualTo("John")
    }

    @Test
    fun `parallel fetch should complete`() = runTest {
        // Given
        coEvery { userRepository.findById(any()) } coAnswers {
            delay(100)  // 仮想時間で即座に進む
            UserEntity(id = firstArg(), name = "User", email = "user@example.com")
        }

        // When
        val start = currentTime
        val results = coroutineScope {
            (1..10).map { id ->
                async { userService.findById(UserId(id.toLong())) }
            }.awaitAll()
        }
        val elapsed = currentTime - start

        // Then
        assertThat(results).hasSize(10)
        assertThat(elapsed).isEqualTo(100)  // 並列実行なので100msのみ
    }
}
```

### Flow のテスト

```kotlin
@Test
fun `streamOrders should emit orders`() = runTest {
    // Given
    val orders = listOf(
        OrderEntity(id = 1, status = OrderStatus.PENDING),
        OrderEntity(id = 2, status = OrderStatus.PENDING),
    )
    every { orderRepository.findByStatus(OrderStatus.PENDING) } returns orders.asFlow()

    // When
    val results = orderService.streamByStatus(OrderStatus.PENDING).toList()

    // Then
    assertThat(results).hasSize(2)
}

@Test
fun `flow should handle errors`() = runTest {
    // Given
    val errorFlow = flow<Data> {
        throw IOException("Network error")
    }.catch { emit(Data.empty()) }

    // When
    val result = errorFlow.first()

    // Then
    assertThat(result).isEqualTo(Data.empty())
}
```

## アンチパターン

### GlobalScope の使用

```kotlin
// NG: GlobalScopeは構造化されていない
GlobalScope.launch {
    processAsync()
}

// OK: 適切なスコープを使用
coroutineScope {
    launch {
        processAsync()
    }
}
```

### runBlocking の乱用

```kotlin
// NG: WebFluxのHandler内でrunBlocking
suspend fun handler(request: ServerRequest): ServerResponse {
    val result = runBlocking {  // イベントループをブロック！
        heavyComputation()
    }
    return ServerResponse.ok().bodyValueAndAwait(result)
}

// OK: withContextで適切なDispatcherを使用
suspend fun handler(request: ServerRequest): ServerResponse {
    val result = withContext(Dispatchers.Default) {
        heavyComputation()
    }
    return ServerResponse.ok().bodyValueAndAwait(result)
}
```

### 例外の握りつぶし

```kotlin
// NG: 例外を握りつぶす
launch {
    try {
        riskyOperation()
    } catch (e: Exception) {
        // 何もしない
    }
}

// OK: 適切にログ出力、再スロー、または結果として返す
launch {
    try {
        riskyOperation()
    } catch (e: Exception) {
        logger.error(e) { "Operation failed" }
        throw e  // または適切な処理
    }
}
```
