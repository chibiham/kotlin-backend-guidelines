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
/api/{version}/{resource}/{id}/{sub-resource}
```

```kotlin
@Configuration
class ApiRouter(
    private val userHandler: UserHandler,
    private val orderHandler: OrderHandler,
) {
    @Bean
    fun apiRoutes(): RouterFunction<ServerResponse> = coRouter {
        "/api/v1".nest {
            // ユーザーリソース
            "/users".nest {
                GET("", userHandler::findAll)
                POST("", userHandler::create)
                GET("/{id}", userHandler::findById)
                PUT("/{id}", userHandler::update)
                DELETE("/{id}", userHandler::delete)

                // サブリソース: ユーザーの注文
                GET("/{id}/orders", orderHandler::findByUserId)
            }

            // 注文リソース
            "/orders".nest {
                GET("", orderHandler::findAll)
                POST("", orderHandler::create)
                GET("/{id}", orderHandler::findById)
                PATCH("/{id}/status", orderHandler::updateStatus)
                POST("/{id}/cancel", orderHandler::cancel)
            }
        }
    }
}
```

### バージョニング

URL パスにバージョンを含める。

```
/api/v1/users
/api/v2/users
```

**バージョンを上げる基準:**

- 既存フィールドの削除や型変更
- 必須フィールドの追加
- レスポンス構造の大幅な変更

**バージョンを上げない変更:**

- オプショナルフィールドの追加
- 新しいエンドポイントの追加
- バグ修正

## HTTP メソッド

### メソッドの使い分け

| メソッド | 用途                         | 冪等性 | 安全性 |
| -------- | ---------------------------- | ------ | ------ |
| GET      | リソースの取得               | ○      | ○      |
| POST     | リソースの作成、その他の操作 | ×      | ×      |
| PUT      | リソースの完全な置換         | ○      | ×      |
| PATCH    | リソースの部分更新           | ○      | ×      |
| DELETE   | リソースの削除               | ○      | ×      |

### 具体例

```kotlin
// GET: リソースの取得
GET /api/v1/users          // ユーザー一覧
GET /api/v1/users/123      // 特定ユーザー

// POST: リソースの作成
POST /api/v1/users         // ユーザー作成

// PUT: リソースの完全な置換（全フィールドを送信）
PUT /api/v1/users/123      // ユーザー情報の置換

// PATCH: リソースの部分更新（変更フィールドのみ送信）
PATCH /api/v1/users/123    // ユーザー情報の部分更新

// DELETE: リソースの削除
DELETE /api/v1/users/123   // ユーザー削除

// POST: リソース作成以外の操作（状態変更など）
POST /api/v1/orders/123/cancel    // 注文キャンセル
POST /api/v1/users/123/activate   // ユーザー有効化
```

## HTTP ステータスコード

### 成功レスポンス

| コード         | 用途               | 使用例                     |
| -------------- | ------------------ | -------------------------- |
| 200 OK         | 一般的な成功       | GET, PUT, PATCH, DELETE    |
| 201 Created    | リソース作成成功   | POST                       |
| 204 No Content | 成功だがボディなし | DELETE（レスポンス不要時） |

### クライアントエラー

| コード                   | 用途                 | 使用例                     |
| ------------------------ | -------------------- | -------------------------- |
| 400 Bad Request          | リクエスト不正       | バリデーションエラー       |
| 401 Unauthorized         | 認証が必要           | 未認証アクセス             |
| 403 Forbidden            | 権限なし             | 認証済みだが権限不足       |
| 404 Not Found            | リソースが存在しない | 存在しない ID へのアクセス |
| 409 Conflict             | 競合                 | 重複登録、楽観ロック失敗   |
| 422 Unprocessable Entity | 処理不可             | ビジネスルール違反         |

### サーバーエラー

| コード                    | 用途               | 使用例                |
| ------------------------- | ------------------ | --------------------- |
| 500 Internal Server Error | サーバー内部エラー | 予期しない例外        |
| 502 Bad Gateway           | 外部サービスエラー | 外部 API 呼び出し失敗 |
| 503 Service Unavailable   | サービス利用不可   | メンテナンス中        |

### 実装例

```kotlin
@Component
class UserHandler(private val userService: UserService) {

    suspend fun create(request: ServerRequest): ServerResponse {
        val req = request.awaitBody<CreateUserRequest>()
        val user = userService.create(req)

        return ServerResponse
            .created(URI("/api/v1/users/${user.id.value}"))
            .bodyValueAndAwait(user.toResponse())
    }

    suspend fun update(request: ServerRequest): ServerResponse {
        val id = UserId(request.pathVariable("id").toLong())
        val req = request.awaitBody<UpdateUserRequest>()
        val user = userService.update(id, req)

        return ServerResponse.ok().bodyValueAndAwait(user.toResponse())
    }

    suspend fun delete(request: ServerRequest): ServerResponse {
        val id = UserId(request.pathVariable("id").toLong())
        userService.delete(id)

        return ServerResponse.noContent().buildAndAwait()
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

// Handler での使用
suspend fun findAll(request: ServerRequest): ServerResponse {
    val criteria = UserSearchCriteria(
        name = request.queryParamOrNull("name"),
        email = request.queryParamOrNull("email"),
        status = request.queryParamOrNull("status")?.let { UserStatus.valueOf(it) },
    )
    val pageRequest = PageRequest(
        page = request.queryParamOrNull("page")?.toIntOrNull() ?: 0,
        size = request.queryParamOrNull("size")?.toIntOrNull() ?: 20,
        sort = request.queryParamOrNull("sort"),
    )

    val result = userService.findAll(criteria, pageRequest)
    return ServerResponse.ok().bodyValueAndAwait(result)
}

// 拡張関数
fun ServerRequest.queryParamOrNull(name: String): String? =
    queryParam(name).orElse(null)
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
suspend fun findAll(request: ServerRequest): ServerResponse {
    val page = userService.findAll(criteria, pageRequest)

    val response = PageResponse(
        content = page.content.map { it.toResponse() },
        page = page.number,
        size = page.size,
        totalElements = page.totalElements,
        totalPages = page.totalPages,
        hasNext = page.hasNext(),
        hasPrevious = page.hasPrevious(),
    )

    return ServerResponse.ok().bodyValueAndAwait(response)
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

// 一覧: GET /api/v1/users -> List<UserSummaryResponse>
// 詳細: GET /api/v1/users/{id} -> UserDetailResponse
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
  "instance": "/api/v1/users",
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
```

## HATEOAS（任意）

必要に応じて HATEOAS リンクを含める。

```kotlin
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    val links: Links,
) {
    data class Links(
        val self: String,
        val orders: String,
        val update: String,
        val delete: String,
    )
}

fun User.toResponse(): UserResponse = UserResponse(
    id = this.id.value,
    name = this.name.value,
    email = this.email.value,
    links = UserResponse.Links(
        self = "/api/v1/users/${this.id.value}",
        orders = "/api/v1/users/${this.id.value}/orders",
        update = "/api/v1/users/${this.id.value}",
        delete = "/api/v1/users/${this.id.value}",
    ),
)
```

```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "links": {
    "self": "/api/v1/users/123",
    "orders": "/api/v1/users/123/orders",
    "update": "/api/v1/users/123",
    "delete": "/api/v1/users/123"
  }
}
```
