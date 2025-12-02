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

    // Not Found (404)
    val USER_NOT_FOUND = URI("$BASE_URI/user-not-found")
    val ORDER_NOT_FOUND = URI("$BASE_URI/order-not-found")

    // Bad Request (400)
    val VALIDATION_ERROR = URI("$BASE_URI/validation-error")          // 入力値の形式エラー専用
    val DUPLICATE_EMAIL = URI("$BASE_URI/duplicate-email")            // ビジネスロジックエラー
    val INSUFFICIENT_BALANCE = URI("$BASE_URI/insufficient-balance")  // ビジネスロジックエラー
    val INSUFFICIENT_STOCK = URI("$BASE_URI/insufficient-stock")      // ビジネスロジックエラー

    // Bad Gateway (502)
    val EXTERNAL_SERVICE_ERROR = URI("$BASE_URI/external-service-error")

    // Internal Server Error (500)
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

        // 入力値の形式エラー専用
        data class Validation(
            val errors: List<FieldError>,
        ) : BadRequest("Validation failed") {
            override val type: URI = ProblemTypes.VALIDATION_ERROR
            override val title: String = "Validation Error"

            data class FieldError(val field: String, val message: String)
        }

        // ビジネスロジックエラー
        data class DuplicateEmail(val email: String) : BadRequest("Email already exists: $email") {
            override val type: URI = ProblemTypes.DUPLICATE_EMAIL
            override val title: String = "Duplicate Email"
        }

        data class InsufficientBalance(
            val required: Long,
            val available: Long,
        ) : BadRequest("Insufficient balance: required $required, available $available") {
            override val type: URI = ProblemTypes.INSUFFICIENT_BALANCE
            override val title: String = "Insufficient Balance"
        }

        data class InsufficientStock(
            val productId: Long,
            val requested: Int,
            val available: Int,
        ) : BadRequest("Insufficient stock for product $productId: requested $requested, available $available") {
            override val type: URI = ProblemTypes.INSUFFICIENT_STOCK
            override val title: String = "Insufficient Stock"
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

    @ExceptionHandler(ServerWebInputException::class)
    fun handleServerWebInputException(
        ex: ServerWebInputException,
        exchange: ServerWebExchange,
    ): Mono<ServerResponse> {
        // Nullableでない値がnullだった場合など、デシリアライゼーションエラー
        val problemDetail = ValidationProblemDetail(
            type = ProblemTypes.VALIDATION_ERROR,
            title = "Validation Error",
            status = HttpStatus.BAD_REQUEST.value(),
            detail = ex.reason ?: "Invalid request body",
            instance = URI(exchange.request.path.value()),
            errors = listOf(
                ValidationProblemDetail.FieldError(
                    field = ex.methodParameter?.parameterName ?: "body",
                    message = ex.reason ?: "Invalid value"
                )
            ),
        )

        return ServerResponse
            .status(HttpStatus.BAD_REQUEST)
            .contentType(MediaType.APPLICATION_PROBLEM_JSON)
            .bodyValue(problemDetail)
    }

    @ExceptionHandler(DecodingException::class)
    fun handleDecodingException(
        ex: DecodingException,
        exchange: ServerWebExchange,
    ): Mono<ServerResponse> {
        // JSONパースエラー（Nullableでない値がnullなど）
        logger.warn(ex) { "Request body decoding error: ${exchange.request.path}" }

        val problemDetail = ProblemDetail(
            type = ProblemTypes.VALIDATION_ERROR,
            title = "Validation Error",
            status = HttpStatus.BAD_REQUEST.value(),
            detail = "Invalid request body format",
            instance = URI(exchange.request.path.value()),
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

        // IDはAuto Incrementでnull（DB採番前）
        val user = User(
            id = null,
            name = request.name,
            email = request.email,
        )
        return userRepository.save(user)
    }
}
```

### Controller での例外処理

Controller では例外をキャッチせず、GlobalExceptionHandler に委ねる。

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {

    // 良い例: 例外をそのまま伝播
    @GetMapping("/{id}")
    suspend fun findById(@PathVariable id: Long): UserResponse {
        val user = userService.findById(id)  // 例外はGlobalExceptionHandlerで処理
        return user.toResponse()
    }

    // 悪い例: Controllerで例外をキャッチ
    @GetMapping("/{id}/bad")
    suspend fun findByIdBad(@PathVariable id: Long): ResponseEntity<UserResponse> {
        return try {
            val user = userService.findById(id)
            ResponseEntity.ok(user.toResponse())
        } catch (e: ApplicationException.NotFound.User) {
            // NG: ここでキャッチすべきではない
            ResponseEntity.notFound().build()
        }
    }
}
```

## バリデーション

### バリデーションとビジネスロジックエラーの区別

入力値のバリデーションとビジネスロジックエラーを明確に区別する。

| 分類 | 実施場所 | エラータイプ | 例 |
|------|---------|-------------|-----|
| **バリデーション**<br>（入力値の形式チェック） | Request DTO（Bean Validation） | `validation-error`（共通） | フォーマット、必須チェック、文字数制限、日付範囲、パスワード一致 |
| **ビジネスロジックエラー**<br>（業務ルール違反） | Application/Domain層 | 個別の`type`を割り当て | メール重複チェック、在庫不足、残高不足、状態遷移不可 |

#### バリデーション（入力値の形式チェック）

入力値のみから判定可能なチェック。Request DTOで実施し、`validation-error`タイプを使用する。

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "Validation failed",
  "errors": [...]
}
```

#### ビジネスロジックエラー（業務ルール違反）

データベースや外部サービスの状態、またはビジネスルールに依存するエラー。Application/Domain層で実施し、個別の`type`を割り当てる。

```json
{
  "type": "https://api.example.com/problems/duplicate-email",
  "title": "Duplicate Email",
  "status": 400,
  "detail": "Email already exists: user@example.com"
}
```

```json
{
  "type": "https://api.example.com/problems/insufficient-stock",
  "title": "Insufficient Stock",
  "status": 400,
  "detail": "Requested quantity exceeds available stock"
}
```

### Nullableでない値がnullだった場合

Kotlinの`@RequestBody`で非Nullable型を定義した場合、nullが送られると以下の例外が発生する：

| 例外 | 発生タイミング | 処理 |
|------|-------------|------|
| `ServerWebInputException` | 非Nullable型の引数にnullが渡された場合 | `handleServerWebInputException`で処理 |
| `DecodingException` | JSONデシリアライズ時にnullが含まれていた場合 | `handleDecodingException`で処理 |

#### リクエスト例

```kotlin
data class CreateUserRequest(
    val name: String,  // 非Nullable
    val email: String, // 非Nullable
)
```

```json
// NG: nameがnull
{
  "name": null,
  "email": "user@example.com"
}
```

#### レスポンス例

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "Invalid request body",
  "instance": "/api/users",
  "errors": [
    {
      "field": "body",
      "message": "Invalid value"
    }
  ]
}
```

**推奨**: 非Nullable型の場合、nullを許容しないため、クライアント側で適切なエラーメッセージを表示する。より詳細なエラー情報が必要な場合は、Bean Validationの`@NotNull`アノテーションを併用する。

```kotlin
data class CreateUserRequest(
    @field:NotNull(message = "Name is required")
    val name: String?,  // Nullableにして@NotNullで検証

    @field:NotNull(message = "Email is required")
    @field:Email(message = "Invalid email format")
    val email: String?,
)
```

### Bean Validation の使用

Request DTOに`@field:`アノテーションでバリデーションルールを定義する。

```kotlin
// presentation/dto/UserRequest.kt
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

Controllerで`@Valid`アノテーションを使用してバリデーションを実行する。

```kotlin
// presentation/UserController.kt
@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun create(@Valid @RequestBody request: CreateUserRequest): UserResponse {
        // @Validアノテーションにより自動的にバリデーションが実行される
        val user = userService.createUser(
            CreateUserCommand(
                name = request.name,
                email = request.email,
                age = request.age
            )
        )

        return user.toResponse()
    }
}
```

**注意**: `@RestController`では`@Valid`アノテーションが自動的に機能するため、手動でのバリデーション実行は不要です。

バリデーションエラーは`GlobalExceptionHandler`で統一的に処理される。

```kotlin
// ApplicationException.BadRequest.Validationがスローされた場合の例
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "Validation failed",
  "instance": "/api/users",
  "errors": [
    {
      "field": "email",
      "message": "Email must be a valid email address"
    },
    {
      "field": "name",
      "message": "Name is required"
    }
  ]
}
```

### クロスフィールドバリデーション

複数フィールドにまたがるバリデーションで、**入力値のみから判定可能な場合**はRequest DTOに記述できる。

```kotlin
// presentation/dto/ReservationRequest.kt
data class CreateReservationRequest(
    @field:NotNull
    val startAt: Instant?,

    @field:NotNull
    val endAt: Instant?,
) {
    // 複数フィールドのバリデーション
    @get:AssertTrue(message = "End time must be after start time")
    val isValidDateRange: Boolean
        get() = startAt != null && endAt != null && endAt.isAfter(startAt)
}
```

より複雑な場合はカスタムバリデーターを使用：

```kotlin
// カスタムアノテーション
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
@Constraint(validatedBy = [DateRangeValidator::class])
annotation class ValidDateRange(
    val message: String = "Invalid date range",
    val groups: Array<KClass<*>> = [],
    val payload: Array<KClass<out Payload>> = []
)

// バリデーター実装
class DateRangeValidator : ConstraintValidator<ValidDateRange, CreateReservationRequest> {
    override fun isValid(value: CreateReservationRequest, context: ConstraintValidatorContext): Boolean {
        if (value.startAt == null || value.endAt == null) {
            return true  // @NotNullで個別にチェックするため、ここではスキップ
        }

        // 終了日時が開始日時より後であること
        if (!value.endAt.isAfter(value.startAt)) {
            context.disableDefaultConstraintViolation()
            context.buildConstraintViolationWithTemplate("End time must be after start time")
                .addPropertyNode("endAt")
                .addConstraintViolation()
            return false
        }

        // 予約期間が30日以内であること
        if (ChronoUnit.DAYS.between(value.startAt, value.endAt) > 30) {
            context.disableDefaultConstraintViolation()
            context.buildConstraintViolationWithTemplate("Reservation period must not exceed 30 days")
                .addPropertyNode("endAt")
                .addConstraintViolation()
            return false
        }

        return true
    }
}

// 使用例
@ValidDateRange
data class CreateReservationRequest(
    @field:NotNull(message = "Start time is required")
    val startAt: Instant?,

    @field:NotNull(message = "End time is required")
    val endAt: Instant?,

    @field:Positive(message = "Number of guests must be positive")
    val numberOfGuests: Int?,
)
```

これらのバリデーションエラーはすべて`validation-error`タイプとして扱われる。

### ビジネスロジックエラーの実装

データベースや外部サービスの状態、またはビジネスルールに依存するエラーはApplication/Domain層で検証し、**個別のExceptionをthrow**する。

```kotlin
// ProblemTypesに追加
object ProblemTypes {
    // ... 既存の定義 ...
    val INSUFFICIENT_STOCK = URI("$BASE_URI/insufficient-stock")
}

// ApplicationExceptionに追加
sealed class ApplicationException {
    // ... 既存の定義 ...

    sealed class BadRequest(message: String) : ApplicationException(message) {
        override val status: HttpStatus = HttpStatus.BAD_REQUEST

        // ビジネスロジックエラー
        data class InsufficientStock(
            val productId: Long,
            val requested: Int,
            val available: Int,
        ) : BadRequest("Insufficient stock for product $productId: requested $requested, available $available") {
            override val type: URI = ProblemTypes.INSUFFICIENT_STOCK
            override val title: String = "Insufficient Stock"
        }
    }
}

// Serviceでの使用例
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val inventoryService: InventoryService,
) {
    suspend fun create(request: CreateOrderRequest): Order {
        // ビジネスロジックチェック: 在庫確認
        request.items.forEach { item ->
            val available = inventoryService.getAvailableQuantity(item.productId)
            if (available < item.quantity) {
                // 個別のExceptionをthrow（バリデーションエラーではない）
                throw ApplicationException.BadRequest.InsufficientStock(
                    productId = item.productId,
                    requested = item.quantity,
                    available = available
                )
            }
        }

        // ビジネスロジックチェック: メール重複確認
        if (userRepository.existsByEmail(request.email)) {
            throw ApplicationException.BadRequest.DuplicateEmail(request.email)
        }

        // 注文作成処理
        ...
    }
}
```

**重要**: `ApplicationException.BadRequest.Validation`は入力値のフォーマットエラー専用。ビジネスロジックエラーには使用しない。

## ロギング

### ログレベルの基準

| 例外の種類           | ログレベル | 理由                                           |
| -------------------- | ---------- | ---------------------------------------------- |
| NotFound             | INFO       | 正常なフロー（存在しないリソースへのアクセス） |
| BadRequest           | INFO       | クライアントの入力エラー・ビジネスルール違反   |
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
