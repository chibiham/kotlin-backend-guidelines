# エラーハンドリング

## 基本方針

1. **RFC 9457 Problem Details**に準拠したエラーレスポンスを返す
2. 例外クラスは**sealed class**で階層化し、網羅的なハンドリングを保証する
3. **@ControllerAdvice**でエラーハンドリングを一元管理する

## RFC 9457 Problem Details

### 概要

RFC 9457（旧 RFC 7807）は、HTTP エラーレスポンスの標準フォーマットを定義している。以下のフィールドを持つ JSON オブジェクトを返す。

| フィールド | 必須 | 説明                                 |
| ---------- | ---- | ------------------------------------ |
| `type`     | 推奨 | 問題の種類を識別する URI             |
| `title`    | 推奨 | 人間が読める短い要約                 |
| `status`   | 推奨 | HTTP ステータスコード                |
| `detail`   | 任意 | 問題の詳細説明                       |
| `instance` | 任意 | 問題が発生した具体的なリソースの URI |

### レスポンス例

```json
{
  "type": "https://example.com/problems/user-not-found",
  "title": "User Not Found",
  "status": 404,
  "detail": "User with ID '12345' was not found",
  "instance": "/api/users/12345"
}
```

### 拡張フィールド

RFC 9457 では独自フィールドの追加が許可されている。

```json
{
  "type": "https://example.com/problems/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "Request validation failed",
  "instance": "/api/users",
  "errors": [
    {
      "field": "email",
      "message": "must be a valid email address"
    },
    {
      "field": "name",
      "message": "must not be blank"
    }
  ]
}
```

## Problem Details の実装

### レスポンスモデル

```kotlin
// 基本のProblem Details
data class ProblemDetail(
    val type: URI,
    val title: String,
    val status: Int,
    val detail: String? = null,
    val instance: URI? = null,
)

// バリデーションエラー用の拡張
data class ValidationProblemDetail(
    val type: URI,
    val title: String,
    val status: Int,
    val detail: String? = null,
    val instance: URI? = null,
    val errors: List<FieldError>,
) {
    data class FieldError(
        val field: String,
        val message: String,
    )
}
```

### 問題タイプの定義

```kotlin
object ProblemTypes {
    private const val BASE_URI = "https://api.example.com/problems"

    val USER_NOT_FOUND = URI("$BASE_URI/user-not-found")
    val ORDER_NOT_FOUND = URI("$BASE_URI/order-not-found")
    val VALIDATION_ERROR = URI("$BASE_URI/validation-error")
    val DUPLICATE_EMAIL = URI("$BASE_URI/duplicate-email")
    val INSUFFICIENT_BALANCE = URI("$BASE_URI/insufficient-balance")
    val EXTERNAL_SERVICE_ERROR = URI("$BASE_URI/external-service-error")
    val INTERNAL_ERROR = URI("$BASE_URI/internal-error")
}
```

## 例外クラスの設計

### sealed class による階層化

```kotlin
sealed class ApplicationException(
    message: String,
    cause: Throwable? = null,
) : RuntimeException(message, cause) {

    abstract val type: URI
    abstract val title: String
    abstract val status: HttpStatus
    open val detail: String? get() = message

    // 404 Not Found
    sealed class NotFound(message: String) : ApplicationException(message) {
        override val status: HttpStatus = HttpStatus.NOT_FOUND

        data class User(val userId: UserId) : NotFound("User not found: ${userId.value}") {
            override val type: URI = ProblemTypes.USER_NOT_FOUND
            override val title: String = "User Not Found"
        }

        data class Order(val orderId: OrderId) : NotFound("Order not found: ${orderId.value}") {
            override val type: URI = ProblemTypes.ORDER_NOT_FOUND
            override val title: String = "Order Not Found"
        }
    }

    // 400 Bad Request
    sealed class BadRequest(message: String) : ApplicationException(message) {
        override val status: HttpStatus = HttpStatus.BAD_REQUEST

        data class Validation(
            val errors: List<FieldError>,
        ) : BadRequest("Validation failed") {
            override val type: URI = ProblemTypes.VALIDATION_ERROR
            override val title: String = "Validation Error"

            data class FieldError(val field: String, val message: String)
        }

        data class DuplicateEmail(val email: Email) : BadRequest("Email already exists: ${email.value}") {
            override val type: URI = ProblemTypes.DUPLICATE_EMAIL
            override val title: String = "Duplicate Email"
        }
    }

    // 409 Conflict
    sealed class Conflict(message: String) : ApplicationException(message) {
        override val status: HttpStatus = HttpStatus.CONFLICT

        data class InsufficientBalance(
            val required: Money,
            val available: Money,
        ) : Conflict("Insufficient balance: required ${required.amount}, available ${available.amount}") {
            override val type: URI = ProblemTypes.INSUFFICIENT_BALANCE
            override val title: String = "Insufficient Balance"
        }
    }

    // 502 Bad Gateway
    sealed class ExternalServiceError(
        message: String,
        cause: Throwable? = null,
    ) : ApplicationException(message, cause) {
        override val status: HttpStatus = HttpStatus.BAD_GATEWAY

        data class PaymentGateway(
            val gatewayMessage: String,
            override val cause: Throwable? = null,
        ) : ExternalServiceError("Payment gateway error: $gatewayMessage", cause) {
            override val type: URI = ProblemTypes.EXTERNAL_SERVICE_ERROR
            override val title: String = "Payment Gateway Error"
        }
    }

    // 500 Internal Server Error
    data class Internal(
        override val cause: Throwable,
    ) : ApplicationException("Internal server error", cause) {
        override val type: URI = ProblemTypes.INTERNAL_ERROR
        override val title: String = "Internal Server Error"
        override val status: HttpStatus = HttpStatus.INTERNAL_SERVER_ERROR
        override val detail: String? = null  // 内部エラーの詳細は隠す
    }
}
```

### sealed class を使う理由

1. **網羅性の保証**: `when`式で全ケースの処理が強制される
2. **型安全性**: 例外の種類ごとに必要なプロパティを定義できる
3. **階層化**: HTTP ステータスコードごとにグループ化できる
4. **拡張性**: 新しい例外タイプの追加が容易

## グローバル例外ハンドラー

### @ControllerAdvice の実装

```kotlin
@ControllerAdvice
class GlobalExceptionHandler {

    private val logger = KotlinLogging.logger {}

    @ExceptionHandler(ApplicationException::class)
    fun handleApplicationException(
        ex: ApplicationException,
        exchange: ServerWebExchange,
    ): Mono<ServerResponse> {
        val problemDetail = when (ex) {
            is ApplicationException.BadRequest.Validation -> ValidationProblemDetail(
                type = ex.type,
                title = ex.title,
                status = ex.status.value(),
                detail = ex.detail,
                instance = URI(exchange.request.path.value()),
                errors = ex.errors.map { ValidationProblemDetail.FieldError(it.field, it.message) },
            )
            else -> ProblemDetail(
                type = ex.type,
                title = ex.title,
                status = ex.status.value(),
                detail = ex.detail,
                instance = URI(exchange.request.path.value()),
            )
        }

        logException(ex, exchange)

        return ServerResponse
            .status(ex.status)
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .bodyValue(problemDetail)
    }

    @ExceptionHandler(WebExchangeBindException::class)
    fun handleValidationException(
        ex: WebExchangeBindException,
        exchange: ServerWebExchange,
    ): Mono<ServerResponse> {
        val errors = ex.bindingResult.fieldErrors.map { fieldError ->
            ValidationProblemDetail.FieldError(
                field = fieldError.field,
                message = fieldError.defaultMessage ?: "Invalid value",
            )
        }

        val problemDetail = ValidationProblemDetail(
            type = ProblemTypes.VALIDATION_ERROR,
            title = "Validation Error",
            status = HttpStatus.BAD_REQUEST.value(),
            detail = "Request validation failed",
            instance = URI(exchange.request.path.value()),
            errors = errors,
        )

        return ServerResponse
            .status(HttpStatus.BAD_REQUEST)
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .bodyValue(problemDetail)
    }

    @ExceptionHandler(Exception::class)
    fun handleUnexpectedException(
        ex: Exception,
        exchange: ServerWebExchange,
    ): Mono<ServerResponse> {
        logger.error(ex) { "Unexpected error occurred: ${exchange.request.path}" }

        val problemDetail = ProblemDetail(
            type = ProblemTypes.INTERNAL_ERROR,
            title = "Internal Server Error",
            status = HttpStatus.INTERNAL_SERVER_ERROR.value(),
            detail = null,  // 予期しないエラーの詳細は隠す
            instance = URI(exchange.request.path.value()),
        )

        return ServerResponse
            .status(HttpStatus.INTERNAL_SERVER_ERROR)
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .bodyValue(problemDetail)
    }

    private fun logException(ex: ApplicationException, exchange: ServerWebExchange) {
        val path = exchange.request.path.value()
        when (ex) {
            is ApplicationException.NotFound -> logger.info { "${ex.title}: $path - ${ex.message}" }
            is ApplicationException.BadRequest -> logger.info { "${ex.title}: $path - ${ex.message}" }
            is ApplicationException.Conflict -> logger.warn { "${ex.title}: $path - ${ex.message}" }
            is ApplicationException.ExternalServiceError -> logger.error(ex) { "${ex.title}: $path" }
            is ApplicationException.Internal -> logger.error(ex) { "Internal error: $path" }
        }
    }
}
```

### Content-Type

エラーレスポンスの Content-Type は`application/problem+json`を使用する。

```kotlin
MediaType.APPLICATION_PROBLEM_JSON  // "application/problem+json"
```

## 例外の投げ方

### Service での例外スロー

```kotlin
@Service
class UserService(
    private val userRepository: UserRepository,
) {
    suspend fun findById(id: UserId): User {
        return userRepository.findById(id)
            ?: throw ApplicationException.NotFound.User(id)
    }

    suspend fun create(request: CreateUserRequest): User {
        if (userRepository.existsByEmail(request.email)) {
            throw ApplicationException.BadRequest.DuplicateEmail(request.email)
        }

        val user = User(
            id = UserId.generate(),
            name = request.name,
            email = request.email,
        )
        return userRepository.save(user)
    }
}
```

### Handler での例外処理

Handler では例外をキャッチせず、GlobalExceptionHandler に委ねる。

```kotlin
@Component
class UserHandler(private val userService: UserService) {

    // 良い例: 例外をそのまま伝播
    suspend fun findById(request: ServerRequest): ServerResponse {
        val id = UserId(request.pathVariable("id").toLong())
        val user = userService.findById(id)  // 例外はGlobalExceptionHandlerで処理
        return ServerResponse.ok().bodyValueAndAwait(user.toResponse())
    }

    // 悪い例: Handlerで例外をキャッチ
    suspend fun findByIdBad(request: ServerRequest): ServerResponse {
        return try {
            val id = UserId(request.pathVariable("id").toLong())
            val user = userService.findById(id)
            ServerResponse.ok().bodyValueAndAwait(user.toResponse())
        } catch (e: ApplicationException.NotFound.User) {
            // NG: ここでキャッチすべきではない
            ServerResponse.notFound().buildAndAwait()
        }
    }
}
```

## バリデーション

### Bean Validation の使用

```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(max = 100, message = "Name must be at most 100 characters")
    val name: String,

    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email must be a valid email address")
    val email: String,

    @field:Min(value = 0, message = "Age must be non-negative")
    @field:Max(value = 150, message = "Age must be at most 150")
    val age: Int?,
)
```

### カスタムバリデーション

複数フィールドにまたがるバリデーションは Service で行う。

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val inventoryService: InventoryService,
) {
    suspend fun create(request: CreateOrderRequest): Order {
        val errors = mutableListOf<ApplicationException.BadRequest.Validation.FieldError>()

        // 在庫チェック
        request.items.forEach { item ->
            val available = inventoryService.getAvailableQuantity(item.productId)
            if (available < item.quantity) {
                errors.add(
                    ApplicationException.BadRequest.Validation.FieldError(
                        field = "items[${item.productId.value}].quantity",
                        message = "Requested quantity exceeds available stock ($available)",
                    )
                )
            }
        }

        if (errors.isNotEmpty()) {
            throw ApplicationException.BadRequest.Validation(errors)
        }

        // 注文作成処理
        ...
    }
}
```

## ロギング

### ログレベルの基準

| 例外の種類           | ログレベル | 理由                                           |
| -------------------- | ---------- | ---------------------------------------------- |
| NotFound             | INFO       | 正常なフロー（存在しないリソースへのアクセス） |
| BadRequest           | INFO       | クライアントの入力エラー                       |
| Conflict             | WARN       | ビジネスルール違反（注意が必要）               |
| ExternalServiceError | ERROR      | 外部システムの障害                             |
| Internal             | ERROR      | 予期しないエラー                               |

### ログ出力例

```kotlin
private fun logException(ex: ApplicationException, exchange: ServerWebExchange) {
    val requestId = exchange.request.headers.getFirst("X-Request-ID") ?: "unknown"
    val path = exchange.request.path.value()
    val method = exchange.request.method

    when (ex) {
        is ApplicationException.NotFound ->
            logger.info { "[$requestId] $method $path - ${ex.title}: ${ex.message}" }
        is ApplicationException.BadRequest ->
            logger.info { "[$requestId] $method $path - ${ex.title}: ${ex.message}" }
        is ApplicationException.Conflict ->
            logger.warn { "[$requestId] $method $path - ${ex.title}: ${ex.message}" }
        is ApplicationException.ExternalServiceError ->
            logger.error(ex) { "[$requestId] $method $path - ${ex.title}" }
        is ApplicationException.Internal ->
            logger.error(ex) { "[$requestId] $method $path - Unexpected error" }
    }
}
```

## テスト

### 例外ハンドリングのテスト

```kotlin
@WebFluxTest
class GlobalExceptionHandlerTest {

    @Autowired
    private lateinit var webTestClient: WebTestClient

    @MockkBean
    private lateinit var userService: UserService

    @Test
    fun `should return 404 with problem detail when user not found`() {
        // Given
        val userId = UserId(12345)
        coEvery { userService.findById(userId) } throws ApplicationException.NotFound.User(userId)

        // When & Then
        webTestClient.get()
            .uri("/api/users/12345")
            .exchange()
            .expectStatus().isNotFound
            .expectHeader().contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .expectBody()
            .jsonPath("$.type").isEqualTo("https://api.example.com/problems/user-not-found")
            .jsonPath("$.title").isEqualTo("User Not Found")
            .jsonPath("$.status").isEqualTo(404)
            .jsonPath("$.detail").isEqualTo("User not found: 12345")
            .jsonPath("$.instance").isEqualTo("/api/users/12345")
    }

    @Test
    fun `should return 400 with validation errors`() {
        // Given
        val errors = listOf(
            ApplicationException.BadRequest.Validation.FieldError("email", "must be a valid email"),
            ApplicationException.BadRequest.Validation.FieldError("name", "must not be blank"),
        )
        coEvery { userService.create(any()) } throws ApplicationException.BadRequest.Validation(errors)

        // When & Then
        webTestClient.post()
            .uri("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue("""{"email": "invalid", "name": ""}""")
            .exchange()
            .expectStatus().isBadRequest
            .expectBody()
            .jsonPath("$.errors.length()").isEqualTo(2)
            .jsonPath("$.errors[0].field").isEqualTo("email")
            .jsonPath("$.errors[1].field").isEqualTo("name")
    }
}
```
