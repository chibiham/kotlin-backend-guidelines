# セキュリティ

## 基本方針

1. **OAuth 2.0 Resource Server**として動作する
2. **Opaque Token**を使用し、トークンイントロスペクションで検証する
3. **RFC 6750**に準拠した Bearer Token エラーレスポンスを返す
4. **Spring Security**でセキュリティを一元管理する
5. **OAuth 2.0 Scope**によりエンドポイントのアクセス制御を行う
6. 機密情報は環境変数または外部シークレット管理サービスで管理する

## 認証・認可の概要

### アーキテクチャ

```
┌──────────────┐     ┌─────────────────┐     ┌──────────────────┐
│   Client     │────▶│  Authorization  │────▶│  Resource Server │
│ (SPA/API等)  │     │     Server      │     │   (この API)      │
└──────────────┘     └─────────────────┘     └──────────────────┘
     │                       │                        │
     │  1. トークン取得         │                        │
     │  (認可コード or          │                        │
     │   client_credentials) │                        │
     │─────────────────────▶│                        │
     │                       │                        │
     │  2. アクセストークン      │                        │
     │  (Opaque Token)      │                        │
     │◀─────────────────────│                        │
     │                       │                        │
     │  3. APIリクエスト (Bearer Token)               │
     │───────────────────────────────────────────────▶│
     │                       │                        │
     │                       │  4. Token Introspection│
     │                       │◀───────────────────────│
     │                       │                        │
     │                       │  5. Token Info         │
     │                       │───────────────────────▶│
     │                       │                        │
     │  6. レスポンス                                   │
     │◀───────────────────────────────────────────────│
```

### 本 API の役割

本 API は**OAuth 2.0 Resource Server**として動作する。

- 認可サーバー（Keycloak、Auth0、Cognito 等）から発行されたOpaque Tokenを受け取る
- トークンイントロスペクションエンドポイントでトークンの有効性を検証する
- トークンに含まれるscopeに基づいてアクセス制御を行う

### サポートする OAuth 2.0 フロー

| フロー | 用途 | クライアント例 |
|--------|------|---------------|
| 認可コードフロー | ユーザー認証が必要 | SPA, Webアプリケーション |
| Client Credentials | サーバー間通信 | バックエンドサービス、バッチ処理 |

**注意**: Resource Serverとして、クライアントがどのフローでトークンを取得したかは意識せず、トークンの有効性とscopeのみを検証する。

## Spring Security 設定

### Resource Server 設定（Opaque Token）

```kotlin
@Configuration
@EnableWebFluxSecurity
class SecurityConfig {

    @Bean
    fun securityFilterChain(http: ServerHttpSecurity): SecurityWebFilterChain {
        return http
            .csrf { it.disable() }  // REST APIのためCSRF無効化
            .cors { it.configurationSource(corsConfigurationSource()) }
            .authorizeExchange { exchanges ->
                exchanges
                    // 公開エンドポイント
                    .pathMatchers("/api/health", "/api/info").permitAll()
                    .pathMatchers("/api-docs/**", "/swagger-ui/**").permitAll()

                    // Scopeベースのアクセス制御
                    .pathMatchers(HttpMethod.GET, "/api/users/**").hasAuthority("SCOPE_users:read")
                    .pathMatchers(HttpMethod.POST, "/api/users/**").hasAuthority("SCOPE_users:write")
                    .pathMatchers(HttpMethod.PUT, "/api/users/**").hasAuthority("SCOPE_users:write")
                    .pathMatchers(HttpMethod.DELETE, "/api/users/**").hasAuthority("SCOPE_users:delete")

                    .pathMatchers(HttpMethod.GET, "/api/orders/**").hasAuthority("SCOPE_orders:read")
                    .pathMatchers(HttpMethod.POST, "/api/orders/**").hasAuthority("SCOPE_orders:write")

                    // その他は認証必須
                    .pathMatchers("/api/**").authenticated()
                    .anyExchange().denyAll()
            }
            .oauth2ResourceServer { oauth2 ->
                oauth2.opaqueToken { }  // Opaque Token使用
                oauth2.authenticationEntryPoint(bearerTokenAuthenticationEntryPoint())
                oauth2.accessDeniedHandler(bearerTokenAccessDeniedHandler())
            }
            .build()
    }

    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource {
        val configuration = CorsConfiguration().apply {
            allowedOrigins = listOf("https://app.example.com")
            allowedMethods = listOf("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            allowedHeaders = listOf("*")
            allowCredentials = true
            maxAge = 3600
        }
        return UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/api/**", configuration)
        }
    }
}
```

### application.yml

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: https://auth.example.com/realms/my-realm/protocol/openid-connect/token/introspect
          client-id: ${OAUTH2_CLIENT_ID}
          client-secret: ${OAUTH2_CLIENT_SECRET}
```

### Scope命名規則

`{resource}:{action}` 形式でscopeを定義する。

| Scope | 説明 | 例 |
|-------|------|-----|
| `users:read` | ユーザー情報の読み取り | GET /api/users |
| `users:write` | ユーザー情報の作成・更新 | POST/PUT /api/users |
| `users:delete` | ユーザー情報の削除 | DELETE /api/users |
| `orders:read` | 注文情報の読み取り | GET /api/orders |
| `orders:write` | 注文情報の作成・更新 | POST/PUT /api/orders |

## RFC 6750 Bearer Token エラーレスポンス

### 概要

RFC 6750 は Bearer Token の使用方法とエラーレスポンスを定義している。認証・認可エラー時は`WWW-Authenticate`ヘッダーを含めて返す。

### エラーコード

| エラーコード         | 説明                                       | HTTP ステータス |
| -------------------- | ------------------------------------------ | --------------- |
| `invalid_request`    | リクエストが不正（必須パラメータ欠落等）   | 400             |
| `invalid_token`      | トークンが無効（期限切れ、失効、形式不正） | 401             |
| `insufficient_scope` | 必要なスコープを持っていない               | 403             |

### WWW-Authenticate ヘッダー形式

```
WWW-Authenticate: Bearer realm="example",
                         error="invalid_token",
                         error_description="The access token expired"
```

### 実装

```kotlin
@Component
class BearerTokenAuthenticationEntryPoint : ServerAuthenticationEntryPoint {

    override fun commence(
        exchange: ServerWebExchange,
        ex: AuthenticationException,
    ): Mono<Void> {
        val response = exchange.response
        response.statusCode = HttpStatus.UNAUTHORIZED

        val wwwAuthenticate = buildWwwAuthenticate(ex)
        response.headers.set(HttpHeaders.WWW_AUTHENTICATE, wwwAuthenticate)
        response.headers.contentType = MediaType.APPLICATION_JSON

        val body = buildErrorBody(exchange, ex)
        val buffer = response.bufferFactory().wrap(body.toByteArray())

        return response.writeWith(Mono.just(buffer))
    }

    private fun buildWwwAuthenticate(ex: AuthenticationException): String {
        val params = mutableListOf("realm=\"api\"")

        when (ex) {
            is OAuth2AuthenticationException -> {
                val error = ex.error
                params.add("error=\"${error.errorCode}\"")
                error.description?.let { params.add("error_description=\"$it\"") }
            }
            else -> {
                params.add("error=\"invalid_token\"")
                params.add("error_description=\"${ex.message ?: "Authentication failed"}\"")
            }
        }

        return "Bearer ${params.joinToString(", ")}"
    }

    private fun buildErrorBody(exchange: ServerWebExchange, ex: AuthenticationException): String {
        val errorCode = when (ex) {
            is OAuth2AuthenticationException -> ex.error.errorCode
            else -> "invalid_token"
        }

        return """
            {
                "type": "https://api.example.com/problems/authentication-error",
                "title": "Authentication Error",
                "status": 401,
                "detail": "${ex.message ?: "Authentication required"}",
                "instance": "${exchange.request.path.value()}",
                "error": "$errorCode"
            }
        """.trimIndent()
    }
}

@Component
class BearerTokenAccessDeniedHandler : ServerAccessDeniedHandler {

    override fun handle(
        exchange: ServerWebExchange,
        denied: AccessDeniedException,
    ): Mono<Void> {
        val response = exchange.response
        response.statusCode = HttpStatus.FORBIDDEN

        val wwwAuthenticate = """Bearer realm="api", error="insufficient_scope", error_description="The request requires higher privileges""""
        response.headers.set(HttpHeaders.WWW_AUTHENTICATE, wwwAuthenticate)
        response.headers.contentType = MediaType.APPLICATION_JSON

        val body = """
            {
                "type": "https://api.example.com/problems/authorization-error",
                "title": "Authorization Error",
                "status": 403,
                "detail": "Insufficient privileges to access this resource",
                "instance": "${exchange.request.path.value()}",
                "error": "insufficient_scope"
            }
        """.trimIndent()

        val buffer = response.bufferFactory().wrap(body.toByteArray())
        return response.writeWith(Mono.just(buffer))
    }
}
```

## Scopeベースのアクセス制御

### 基本方針

**アクセス制御はSecurityConfigでScopeベースに行い、ServiceやController層では権限チェックを行わない。**

- ✅ SecurityConfigで`hasAuthority("SCOPE_xxx")`を使用
- ❌ Serviceで`@PreAuthorize`を使用しない
- ❌ Controllerで手動の権限チェックを行わない

### 理由

1. **責務の分離**: セキュリティ設定は一箇所に集約
2. **テスト容易性**: ビジネスロジックとセキュリティロジックが分離される
3. **保守性**: セキュリティポリシーの変更が容易

### トークン情報の取得

```kotlin
// Controllerでトークン情報を取得
@RestController
@RequestMapping("/api/orders")
class OrderController(private val orderService: OrderService) {

    @GetMapping("/my")
    suspend fun findMyOrders(authentication: BearerTokenAuthentication): List<OrderResponse> {
        // Spring Securityが自動的にBearerTokenAuthenticationを注入
        val userId = authentication.tokenAttributes["sub"]?.toString()?.toLongOrNull()
            ?: throw ApplicationException.Unauthorized("Invalid token: missing subject")

        val orders = orderService.findByUserId(userId)
        return orders.map { it.toResponse() }
    }

    @GetMapping("/my/email")
    suspend fun getCurrentUserEmail(authentication: BearerTokenAuthentication?): String? {
        // Optional injection: 認証が不要なエンドポイントで使用
        return authentication?.tokenAttributes?.get("email")?.toString()
    }
}
```

### トークンイントロスペクションレスポンス例

```json
{
  "active": true,
  "scope": "users:read users:write orders:read",
  "client_id": "my-client",
  "username": "john.doe",
  "token_type": "Bearer",
  "exp": 1704067200,
  "iat": 1704063600,
  "sub": "12345",
  "email": "user@example.com"
}
```

## セキュリティヘッダー

### レスポンスヘッダーの設定

```kotlin
@Component
class SecurityHeadersFilter : WebFilter {

    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        exchange.response.headers.apply {
            // XSS対策
            set("X-Content-Type-Options", "nosniff")
            set("X-Frame-Options", "DENY")
            set("X-XSS-Protection", "1; mode=block")

            // HTTPSの強制
            set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")

            // コンテンツセキュリティポリシー
            set("Content-Security-Policy", "default-src 'self'")

            // リファラーポリシー
            set("Referrer-Policy", "strict-origin-when-cross-origin")

            // 権限ポリシー
            set("Permissions-Policy", "geolocation=(), microphone=(), camera=()")
        }

        return chain.filter(exchange)
    }
}
```

## 入力バリデーション（セキュリティ観点）

### SQL インジェクション対策

R2DBC や Spring Data R2DBC を使用し、パラメータバインディングを必ず使用する。

```kotlin
// 推奨: パラメータバインディング
@Repository
interface UserRepository : CoroutineCrudRepository<UserEntity, Long> {
    suspend fun findByEmail(email: String): UserEntity?
}

// 非推奨: 文字列連結（絶対に使用しない）
// "SELECT * FROM users WHERE email = '$email'"
```

### XSS 対策

```kotlin
// HTMLエスケープが必要な場合
fun String.escapeHtml(): String =
    this.replace("&", "&amp;")
        .replace("<", "&lt;")
        .replace(">", "&gt;")
        .replace("\"", "&quot;")
        .replace("'", "&#x27;")

// 入力値のサニタイズ
data class CommentRequest(
    @field:NotBlank
    @field:Size(max = 1000)
    val content: String,
) {
    fun sanitized(): CommentRequest = copy(
        content = content.escapeHtml()
    )
}
```

### パストラバーサル対策

```kotlin
fun validateFilePath(filename: String): String {
    // パストラバーサル文字を検出
    require(!filename.contains("..")) { "Invalid filename" }
    require(!filename.contains("/")) { "Invalid filename" }
    require(!filename.contains("\\")) { "Invalid filename" }

    // ホワイトリストで許可する文字を制限
    require(filename.matches(Regex("^[a-zA-Z0-9._-]+$"))) { "Invalid filename" }

    return filename
}
```

## 機密情報の管理

### 環境変数の使用

```yaml
# application.yml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${OAUTH2_ISSUER_URI}

app:
  encryption:
    secret-key: ${ENCRYPTION_SECRET_KEY}
```

### 機密情報をログに出力しない

```kotlin
// 非推奨: 機密情報をログ出力
logger.info { "User authenticated with token: $accessToken" }

// 推奨: マスキングまたは出力しない
logger.info { "User authenticated: userId=${userId}" }

// data classのtoString()から機密情報を除外
data class UserCredentials(
    val username: String,
    private val password: String,
) {
    override fun toString(): String = "UserCredentials(username=$username, password=***)"
}
```

## テスト

### セキュリティテスト

```kotlin
@WebFluxTest
@Import(SecurityConfig::class)
class SecurityTest {

    @Autowired
    private lateinit var webTestClient: WebTestClient

    @Test
    fun `should return 401 when no token provided`() {
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
    fun `should return 200 when authenticated with correct scope`() {
        webTestClient.get()
            .uri("/api/users")
            .headers { it.setBearerAuth(createMockToken("users:read")) }
            .exchange()
            .expectStatus().isOk
    }

    @Test
    fun `should return 403 when insufficient scope`() {
        webTestClient.delete()
            .uri("/api/users/123")
            .headers { it.setBearerAuth(createMockToken("users:read")) }  // users:delete が必要
            .exchange()
            .expectStatus().isForbidden
    }

    private fun createMockToken(scope: String): String {
        // テスト用のトークン生成（実際の実装は認可サーバーのモックに依存）
        return "mock-token-with-scope-$scope"
    }
}
```

### Opaque Token モックの作成

```kotlin
@TestConfiguration
class TestSecurityConfig {

    @Bean
    fun opaqueTokenIntrospector(): ReactiveOpaqueTokenIntrospector {
        return ReactiveOpaqueTokenIntrospector { token ->
            // トークンに応じたモック情報を返す
            val attributes = when {
                token.contains("users:read") -> mapOf(
                    "active" to true,
                    "scope" to "users:read",
                    "sub" to "12345",
                    "client_id" to "test-client",
                    "email" to "test@example.com"
                )
                token.contains("users:write") -> mapOf(
                    "active" to true,
                    "scope" to "users:read users:write",
                    "sub" to "12345",
                    "client_id" to "test-client"
                )
                else -> mapOf("active" to false)
            }

            Mono.just(
                OAuth2IntrospectionAuthenticatedPrincipal(
                    attributes["sub"] as? String ?: "unknown",
                    attributes,
                    listOf(SimpleGrantedAuthority("SCOPE_${attributes["scope"]}"))
                )
            )
        }
    }
}
```
