# API 設計

## 基本方針

1. **RESTful**の原則に従う
2. リソース指向で設計し、URL はリソースを表す名詞を使用する
3. リクエスト/レスポンスの DTO を明確に分離する
4. 一貫性のあるレスポンス形式を維持する

## URL 設計

### 基本ルール

| ルール                     | 良い例               | 悪い例                            |
| -------------------------- | -------------------- | --------------------------------- |
| 名詞を使用（動詞は避ける） | `/users`             | `/getUsers`                       |
| 複数形を使用               | `/users`             | `/user`                           |
| 小文字とハイフンを使用     | `/user-profiles`     | `/userProfiles`, `/user_profiles` |
| リソースの階層を表現       | `/users/{id}/orders` | `/user-orders/{userId}`           |

### URL 構造

```
/api/{resource}/{id}/{sub-resource}
```

```kotlin
// ユーザーリソース
@RestController
@RequestMapping("/api/users")
class UserController(
    private val userService: UserService,
) {
    @GetMapping
    suspend fun findAll(): List<UserResponse> = userService.findAll().map { it.toResponse() }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun create(@Valid @RequestBody request: CreateUserRequest): UserResponse {
        val user = userService.create(request)
        return user.toResponse()
    }

    @GetMapping("/{id}")
    suspend fun findById(@PathVariable id: Long): UserResponse {
        val user = userService.findById(id) ?: throw UserNotFoundException(id)
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

    // サブリソース: ユーザーの注文
    @GetMapping("/{id}/orders")
    suspend fun findOrders(@PathVariable id: Long): List<OrderResponse> {
        return orderService.findByUserId(id).map { it.toResponse() }
    }
}

// 注文リソース
@RestController
@RequestMapping("/api/orders")
class OrderController(
    private val orderService: OrderService,
) {
    @GetMapping
    suspend fun findAll(): List<OrderResponse> = orderService.findAll().map { it.toResponse() }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun create(@Valid @RequestBody request: CreateOrderRequest): OrderResponse {
        val order = orderService.create(request)
        return order.toResponse()
    }

    @GetMapping("/{id}")
    suspend fun findById(@PathVariable id: Long): OrderResponse {
        val order = orderService.findById(id) ?: throw OrderNotFoundException(id)
        return order.toResponse()
    }

    @PatchMapping("/{id}/status")
    suspend fun updateStatus(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateOrderStatusRequest
    ): OrderResponse {
        val order = orderService.updateStatus(id, request.status)
        return order.toResponse()
    }

    @PostMapping("/{id}/cancel")
    suspend fun cancel(@PathVariable id: Long): OrderResponse {
        val order = orderService.cancel(id)
        return order.toResponse()
    }
}
```

### バージョニング

**現在はバージョニングを行わない方針**とする。将来的に破壊的変更が必要になった場合に導入を検討する。

#### バージョニングが必要になるケース

以下のような破壊的変更が必要になった場合、バージョニングの導入を検討する：

- 既存フィールドの削除や型変更
- 必須フィールドの追加
- レスポンス構造の大幅な変更
- エンドポイントのパス変更

#### バージョニング導入時の実装例

URLパスにバージョンを含める方式を採用する。

```kotlin
// v1: 既存のAPIをそのまま維持
@RestController
@RequestMapping("/api/v1/users")
class UserControllerV1(private val userService: UserService) {
    @GetMapping
    suspend fun findAll(): List<UserResponse> {
        return userService.findAll().map { it.toResponse() }
    }
    // ... 既存の実装
}

// v2: 新しいAPI
@RestController
@RequestMapping("/api/v2/users")
class UserControllerV2(private val userServiceV2: UserServiceV2) {
    @GetMapping
    suspend fun findAll(): List<UserResponseV2> {
        return userServiceV2.findAll().map { it.toResponseV2() }
    }
    // ... 新しい実装
}
```

#### バージョニング導入時の移行戦略

1. **既存エンドポイントの維持**
   - `/api/users` は `/api/v1/users` にリダイレクト
   - 一定期間（例: 6ヶ月）は両方のパスをサポート

2. **段階的な移行**
   ```kotlin
   // 既存パスからv1へのリダイレクト
   @RestController
   @RequestMapping("/api/users")
   class UserRedirectController {
       @GetMapping
       suspend fun redirectToV1(): ResponseEntity<Void> {
           return ResponseEntity
               .status(HttpStatus.MOVED_PERMANENTLY)
               .location(URI("/api/v1/users"))
               .build()
       }
   }
   ```

3. **クライアントへの通知**
   - レスポンスヘッダーで非推奨を通知
   ```kotlin
   @GetMapping
   suspend fun findAll(response: ServerHttpResponse): List<UserResponse> {
       response.headers.add("Deprecation", "true")
       response.headers.add("Sunset", "2024-12-31")
       response.headers.add("Link", "</api/v1/users>; rel=\"alternate\"")
       return userService.findAll().map { it.toResponse() }
   }
   ```

#### バージョンを上げない変更

以下の変更は既存のエンドポイントに対して行える（破壊的変更ではない）：

- オプショナルフィールドの追加
- 新しいエンドポイントの追加
- バグ修正
- パフォーマンス改善

## HTTP メソッド

### メソッドの使い分け

| メソッド | 用途                         | 冪等性 | 安全性 |
| -------- | ---------------------------- | ------ | ------ |
| GET      | リソースの取得               | ○      | ○      |
| POST     | リソースの作成、その他の操作 | ×      | ×      |
| PUT      | リソースの完全な置換         | ○      | ×      |
| PATCH    | リソースの部分更新           | ○      | ×      |
| DELETE   | リソースの削除               | ○      | ×      |

**冪等性（Idempotent）**: 同じリクエストを複数回実行しても、結果が同じになる性質。ネットワークエラー時のリトライが安全に行える。

**安全性（Safe）**: リソースの状態を変更しない性質。キャッシュやプリフェッチが可能。

```kotlin
// 安全（Safe）: 何度実行してもリソースは変わらない
GET /api/users/123  // データベースは変更されない

// 冪等（Idempotent）: 複数回実行しても結果は同じ
PUT /api/users/123   // 1回でも10回でも最終的な状態は同じ
DELETE /api/users/123  // 1回目は削除、2回目以降は404だが副作用なし

// 非冪等（Non-Idempotent）: 実行するたびに結果が変わる
POST /api/users  // 実行するたびに新しいユーザーが作成される
```

### 具体例

```kotlin
// GET: リソースの取得
GET /api/users          // ユーザー一覧
GET /api/users/123      // 特定ユーザー

// POST: リソースの作成
POST /api/users         // ユーザー作成

// PUT: リソースの完全な置換（全フィールドを送信）
PUT /api/users/123      // ユーザー情報の置換

// PATCH: リソースの部分更新（変更フィールドのみ送信）
PATCH /api/users/123    // ユーザー情報の部分更新

// DELETE: リソースの削除
DELETE /api/users/123   // ユーザー削除

// POST: リソース作成以外の操作（状態変更など）
POST /api/orders/123/cancel    // 注文キャンセル
POST /api/users/123/activate   // ユーザー有効化
```

## HTTP ステータスコード

### 成功レスポンス

| コード         | 用途               | 使用例                     |
| -------------- | ------------------ | -------------------------- |
| 200 OK         | 一般的な成功       | GET, PUT, PATCH, DELETE    |
| 201 Created    | リソース作成成功   | POST                       |
| 204 No Content | 成功だがボディなし | DELETE（レスポンス不要時） |

### クライアントエラー

| コード           | 用途                 | 使用例                                                           |
| ---------------- | -------------------- | ---------------------------------------------------------------- |
| 400 Bad Request  | リクエスト不正       | バリデーションエラー、ビジネスルール違反（重複登録、在庫不足等） |
| 401 Unauthorized | 認証が必要           | 未認証アクセス                                                   |
| 403 Forbidden    | 権限なし             | 認証済みだが権限不足                                             |
| 404 Not Found    | リソースが存在しない | 存在しない ID へのアクセス                                       |

**注意:** 409 Conflictと422 Unprocessable Entityは使用せず、ビジネスルール違反は400 Bad Requestで統一する。個別の`type`フィールドでエラーを区別する。

### サーバーエラー

| コード                    | 用途               | 使用例                |
| ------------------------- | ------------------ | --------------------- |
| 500 Internal Server Error | サーバー内部エラー | 予期しない例外        |
| 502 Bad Gateway           | 外部サービスエラー | 外部 API 呼び出し失敗 |
| 503 Service Unavailable   | サービス利用不可   | メンテナンス中        |

### 実装例

```kotlin
@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun create(@Valid @RequestBody request: CreateUserRequest): UserResponse {
        val user = userService.create(request)
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

## リクエスト設計

### DTO の命名規則

```kotlin
// 作成リクエスト
data class CreateUserRequest(...)
data class CreateOrderRequest(...)

// 更新リクエスト（完全置換）
data class UpdateUserRequest(...)

// 部分更新リクエスト
data class PatchUserRequest(...)

// 検索条件
data class UserSearchCriteria(...)
```

### リクエスト DTO

```kotlin
// 作成リクエスト: 必須フィールドのみ
data class CreateUserRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(max = 100, message = "Name must be at most 100 characters")
    val name: String,

    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Email must be valid")
    val email: String,
)

// 更新リクエスト（PUT）: 全フィールドを含む
data class UpdateUserRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(max = 100)
    val name: String,

    @field:NotBlank(message = "Email is required")
    @field:Email
    val email: String,

    val bio: String?,
)

// 部分更新リクエスト（PATCH）: 全フィールドがnullable
data class PatchUserRequest(
    @field:Size(max = 100)
    val name: String? = null,

    @field:Email
    val email: String? = null,

    val bio: String? = null,
)
```

### クエリパラメータ

```kotlin
// 検索・フィルタリング条件
data class UserSearchCriteria(
    val name: String? = null,
    val email: String? = null,
    val status: UserStatus? = null,
    val createdFrom: LocalDate? = null,
    val createdTo: LocalDate? = null,
)

// ページネーション
data class PageRequest(
    val page: Int = 0,
    val size: Int = 20,
    val sort: String? = null,
)

// Controller での使用
@GetMapping
suspend fun findAll(
    @RequestParam(required = false) name: String?,
    @RequestParam(required = false) email: String?,
    @RequestParam(required = false) status: UserStatus?,
    @RequestParam(required = false, defaultValue = "0") page: Int,
    @RequestParam(required = false, defaultValue = "20") size: Int,
    @RequestParam(required = false) sort: String?,
): PageResponse<UserResponse> {
    val criteria = UserSearchCriteria(
        name = name,
        email = email,
        status = status,
    )
    val pageRequest = PageRequest(
        page = page,
        size = size,
        sort = sort,
    )

    val result = userService.findAll(criteria, pageRequest)
    return result.map { it.toResponse() }
}
```

## レスポンス設計

### 単一リソースのレスポンス

```kotlin
// レスポンスDTO
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    val status: String,
    val createdAt: Instant,
    val updatedAt: Instant,
)

// ドメインモデルからの変換
fun User.toResponse(): UserResponse = UserResponse(
    id = this.id.value,
    name = this.name.value,
    email = this.email.value,
    status = this.status.name,
    createdAt = this.createdAt,
    updatedAt = this.updatedAt,
)
```

```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "status": "ACTIVE",
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z"
}
```

### コレクションのレスポンス

```kotlin
// ページネーション付きレスポンス
data class PageResponse<T>(
    val content: List<T>,
    val page: Int,
    val size: Int,
    val totalElements: Long,
    val totalPages: Int,
    val hasNext: Boolean,
    val hasPrevious: Boolean,
)

// 使用例
@GetMapping
suspend fun findAll(
    @RequestParam(required = false, defaultValue = "0") page: Int,
    @RequestParam(required = false, defaultValue = "20") size: Int,
): PageResponse<UserResponse> {
    val pageResult = userService.findAll(page, size)

    return PageResponse(
        content = pageResult.content.map { it.toResponse() },
        page = pageResult.number,
        size = pageResult.size,
        totalElements = pageResult.totalElements,
        totalPages = pageResult.totalPages,
        hasNext = pageResult.hasNext(),
        hasPrevious = pageResult.hasPrevious(),
    )
}
```

```json
{
  "content": [
    { "id": 1, "name": "Alice", ... },
    { "id": 2, "name": "Bob", ... }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 42,
  "totalPages": 3,
  "hasNext": true,
  "hasPrevious": false
}
```

### ネストしたリソース

```kotlin
// 関連リソースを含むレスポンス
data class OrderResponse(
    val id: Long,
    val status: String,
    val totalAmount: BigDecimal,
    val items: List<OrderItemResponse>,
    val shippingAddress: AddressResponse,
    val createdAt: Instant,
)

data class OrderItemResponse(
    val productId: Long,
    val productName: String,
    val quantity: Int,
    val unitPrice: BigDecimal,
)

data class AddressResponse(
    val street: String,
    val city: String,
    val postalCode: String,
    val country: String,
)
```

### 軽量レスポンス

一覧取得時は軽量なレスポンスを返し、詳細取得時にフルのレスポンスを返す。

```kotlin
// 一覧用（軽量）
data class UserSummaryResponse(
    val id: Long,
    val name: String,
    val email: String,
)

// 詳細用（フル）
data class UserDetailResponse(
    val id: Long,
    val name: String,
    val email: String,
    val bio: String?,
    val status: String,
    val roles: List<String>,
    val createdAt: Instant,
    val updatedAt: Instant,
    val lastLoginAt: Instant?,
)

// 一覧: GET /api/users -> List<UserSummaryResponse>
// 詳細: GET /api/users/{id} -> UserDetailResponse
```

## 日時のフォーマット

### ISO 8601 形式を使用

```kotlin
// レスポンスの日時はISO 8601形式
{
  "createdAt": "2024-01-15T10:30:00Z",
  "scheduledAt": "2024-01-20T15:00:00+09:00"
}

// リクエストでも同形式を受け付ける
data class ScheduleRequest(
    @field:NotNull
    val scheduledAt: Instant,
)
```

### Jackson 設定

```kotlin
@Configuration
class JacksonConfig {

    @Bean
    fun objectMapper(): ObjectMapper = jacksonObjectMapper().apply {
        registerModule(JavaTimeModule())
        disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
        setSerializationInclusion(JsonInclude.Include.NON_NULL)
    }
}
```

## 命名規則

### JSON フィールド

**camelCase を使用する。**

```json
{
  "userId": 123,
  "firstName": "John",
  "lastName": "Doe",
  "emailAddress": "john@example.com",
  "createdAt": "2024-01-15T10:30:00Z"
}
```

### 一貫性のある命名

| 概念       | 命名                       |
| ---------- | -------------------------- |
| 識別子     | `id`, `userId`, `orderId`  |
| 作成日時   | `createdAt`                |
| 更新日時   | `updatedAt`                |
| 削除日時   | `deletedAt`                |
| ステータス | `status`                   |
| 種別       | `type`                     |
| 数量       | `quantity`, `count`        |
| 金額       | `amount`, `price`, `total` |

## エラーレスポンス

エラーレスポンスは RFC 9457 Problem Details に従う。詳細は[04-error-handling.md](./04-error-handling.md)を参照。

```json
{
  "type": "https://api.example.com/problems/validation-error",
  "title": "Validation Error",
  "status": 400,
  "detail": "Request validation failed",
  "instance": "/api/users",
  "errors": [{ "field": "email", "message": "must be a valid email address" }]
}
```

## API ドキュメント

### OpenAPI (Swagger)の使用

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springdoc:springdoc-openapi-starter-webflux-ui:2.3.0")
}

// application.yml
springdoc:
  api-docs:
    path: /api-docs
  swagger-ui:
    path: /swagger-ui.html
```

### アノテーションによるドキュメント

```kotlin
@Schema(description = "ユーザー作成リクエスト")
data class CreateUserRequest(
    @field:Schema(description = "ユーザー名", example = "John Doe", maxLength = 100)
    @field:NotBlank
    val name: String,

    @field:Schema(description = "メールアドレス", example = "john@example.com")
    @field:Email
    val email: String,
)

@Schema(description = "ユーザーレスポンス")
data class UserResponse(
    @field:Schema(description = "ユーザーID", example = "123")
    val id: Long,

    @field:Schema(description = "ユーザー名", example = "John Doe")
    val name: String,

    @field:Schema(description = "ステータス", example = "ACTIVE", allowableValues = ["ACTIVE", "INACTIVE", "SUSPENDED"])
    val status: String,
)
