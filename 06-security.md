# セキュリティ

## 基本方針

1. **OAuth 2.0 + PKCE**による認可フローを採用する
2. **RFC 6750**に準拠した Bearer Token エラーレスポンスを返す
3. **Spring Security**でセキュリティを一元管理する
4. 機密情報は環境変数または外部シークレット管理サービスで管理する

## 認証・認可の概要

### アーキテクチャ

```
┌─────────┐     ┌─────────────────┐     ┌──────────────────┐
│ Client  │────▶│  Authorization  │────▶│  Resource Server │
│  (SPA)  │     │     Server      │     │   (この API)      │
└─────────┘     └─────────────────┘     └──────────────────┘
     │                  │                        │
     │  1. 認可リクエスト  │                        │
     │  (PKCE)          │                        │
     │─────────────────▶│                        │
     │                  │                        │
     │  2. 認可コード     │                        │
     │◀─────────────────│                        │
     │                  │                        │
     │  3. トークン交換   │                        │
     │─────────────────▶│                        │
     │                  │                        │
     │  4. アクセストークン │                        │
     │◀─────────────────│                        │
     │                  │                        │
     │  5. APIリクエスト (Bearer Token)            │
     │────────────────────────────────────────────▶│
     │                                            │
     │  6. レスポンス                               │
     │◀────────────────────────────────────────────│
```

### 本 API の役割

本 API は**Resource Server**として動作する。認可サーバー（Keycloak、Auth0、Cognito 等）から発行されたアクセストークンを検証し、リソースへのアクセスを制御する。

## PKCE (Proof Key for Code Exchange)

### PKCE とは

PKCE は認可コード横取り攻撃を防ぐための OAuth 2.0 拡張（RFC 7636）。パブリッククライアント（SPA、モバイルアプリ）で必須。

### フロー

```
1. クライアント: code_verifier（ランダム文字列）を生成
2. クライアント: code_challenge = SHA256(code_verifier) をBase64URLエンコード
3. クライアント: 認可リクエストに code_challenge を含める
4. 認可サーバー: 認可コードを発行
5. クライアント: トークンリクエストに code_verifier を含める
6. 認可サーバー: code_verifierをハッシュ化し、保存したcode_challengeと比較
7. 認可サーバー: 一致すればトークンを発行
```

### クライアント側の実装例（参考）

```javascript
// code_verifier の生成
function generateCodeVerifier() {
  const array = new Uint8Array(32);
  crypto.getRandomValues(array);
  return base64UrlEncode(array);
}

// code_challenge の生成
async function generateCodeChallenge(verifier) {
  const encoder = new TextEncoder();
  const data = encoder.encode(verifier);
  const hash = await crypto.subtle.digest("SHA-256", data);
  return base64UrlEncode(new Uint8Array(hash));
}

// 認可リクエスト
const authUrl = new URL("https://auth.example.com/authorize");
authUrl.searchParams.set("response_type", "code");
authUrl.searchParams.set("client_id", CLIENT_ID);
authUrl.searchParams.set("redirect_uri", REDIRECT_URI);
authUrl.searchParams.set("scope", "openid profile email");
authUrl.searchParams.set("code_challenge", codeChallenge);
authUrl.searchParams.set("code_challenge_method", "S256");
```

## Spring Security 設定

### Resource Server 設定

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
                    .pathMatchers("/api/v1/health", "/api/v1/info").permitAll()
                    .pathMatchers("/api-docs/**", "/swagger-ui/**").permitAll()
                    // 認証が必要なエンドポイント
                    .pathMatchers("/api/v1/**").authenticated()
                    .anyExchange().denyAll()
            }
            .oauth2ResourceServer { oauth2 ->
                oauth2.jwt { jwt ->
                    jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())
                }
                oauth2.authenticationEntryPoint(bearerTokenAuthenticationEntryPoint())
                oauth2.accessDeniedHandler(bearerTokenAccessDeniedHandler())
            }
            .build()
    }

    @Bean
    fun jwtAuthenticationConverter(): ReactiveJwtAuthenticationConverter {
        val converter = ReactiveJwtAuthenticationConverter()
        converter.setJwtGrantedAuthoritiesConverter(jwtGrantedAuthoritiesConverter())
        return converter
    }

    @Bean
    fun jwtGrantedAuthoritiesConverter(): Converter<Jwt, Flux<GrantedAuthority>> {
        return Converter { jwt ->
            // Keycloakの場合: realm_access.roles からロールを抽出
            val realmAccess = jwt.getClaimAsMap("realm_access")
            val roles = (realmAccess?.get("roles") as? List<*>)
                ?.filterIsInstance<String>()
                ?: emptyList()

            Flux.fromIterable(roles.map { SimpleGrantedAuthority("ROLE_$it") })
        }
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
        jwt:
          issuer-uri: https://auth.example.com/realms/my-realm
          # または jwk-set-uri を直接指定
          # jwk-set-uri: https://auth.example.com/realms/my-realm/protocol/openid-connect/certs
```

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

## ロールベースアクセス制御

### Handler での権限チェック

```kotlin
@Component
class AdminUserHandler(
    private val userService: UserService,
) {
    suspend fun deleteUser(request: ServerRequest): ServerResponse {
        val principal = request.awaitPrincipal()
            ?: throw ApplicationException.Unauthorized("Authentication required")

        val authorities = (principal as JwtAuthenticationToken).authorities
        if (authorities.none { it.authority == "ROLE_ADMIN" }) {
            throw ApplicationException.Forbidden("Admin role required")
        }

        val id = UserId(request.pathVariable("id").toLong())
        userService.delete(id)

        return ServerResponse.noContent().buildAndAwait()
    }
}
```

### メソッドセキュリティ

```kotlin
@Configuration
@EnableReactiveMethodSecurity
class MethodSecurityConfig

@Service
class UserService(private val userRepository: UserRepository) {

    @PreAuthorize("hasRole('ADMIN')")
    suspend fun deleteUser(id: UserId) {
        userRepository.deleteById(id)
    }

    @PreAuthorize("hasRole('USER') or hasRole('ADMIN')")
    suspend fun findById(id: UserId): User? {
        return userRepository.findById(id)
    }

    @PreAuthorize("#userId == authentication.token.claims['sub']")
    suspend fun updateProfile(userId: UserId, request: UpdateProfileRequest): User {
        // 自分自身のプロフィールのみ更新可能
        val user = userRepository.findById(userId)
            ?: throw ApplicationException.NotFound.User(userId)
        return userRepository.save(user.updateProfile(request))
    }
}
```

### カスタム権限チェック

```kotlin
@Component
class ResourceAccessChecker(
    private val orderRepository: OrderRepository,
) {
    suspend fun canAccessOrder(orderId: OrderId, principal: Principal): Boolean {
        val jwt = (principal as JwtAuthenticationToken).token
        val userId = UserId(jwt.subject.toLong())

        val order = orderRepository.findById(orderId) ?: return false
        return order.userId == userId
    }
}

// Serviceでの使用
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val accessChecker: ResourceAccessChecker,
) {
    suspend fun findById(orderId: OrderId, principal: Principal): Order {
        if (!accessChecker.canAccessOrder(orderId, principal)) {
            throw ApplicationException.Forbidden("Cannot access this order")
        }
        return orderRepository.findById(orderId)
            ?: throw ApplicationException.NotFound.Order(orderId)
    }
}
```

## JWT クレームの取得

### 認証情報のユーティリティ

```kotlin
// 拡張関数でJWT情報を取得しやすくする
suspend fun ServerRequest.jwtOrNull(): Jwt? {
    return awaitPrincipal()?.let { (it as? JwtAuthenticationToken)?.token }
}

suspend fun ServerRequest.jwt(): Jwt {
    return jwtOrNull() ?: throw ApplicationException.Unauthorized("Authentication required")
}

suspend fun ServerRequest.currentUserId(): UserId {
    return UserId(jwt().subject.toLong())
}

// Handler での使用
@Component
class OrderHandler(private val orderService: OrderService) {

    suspend fun findMyOrders(request: ServerRequest): ServerResponse {
        val userId = request.currentUserId()
        val orders = orderService.findByUserId(userId)
        return ServerResponse.ok().bodyValueAndAwait(orders.map { it.toResponse() })
    }
}
```

### JWT クレームの構造例

```json
{
  "sub": "12345",
  "iss": "https://auth.example.com/realms/my-realm",
  "aud": "my-api",
  "exp": 1704067200,
  "iat": 1704063600,
  "email": "user@example.com",
  "name": "John Doe",
  "realm_access": {
    "roles": ["user", "admin"]
  },
  "scope": "openid profile email"
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
            .uri("/api/v1/users")
            .exchange()
            .expectStatus().isUnauthorized
            .expectHeader().valueMatches(
                HttpHeaders.WWW_AUTHENTICATE,
                """Bearer realm="api".*"""
            )
    }

    @Test
    @WithMockUser(roles = ["USER"])
    fun `should return 200 when authenticated`() {
        webTestClient.get()
            .uri("/api/v1/users/me")
            .exchange()
            .expectStatus().isOk
    }

    @Test
    @WithMockUser(roles = ["USER"])
    fun `should return 403 when insufficient role`() {
        webTestClient.delete()
            .uri("/api/v1/admin/users/123")
            .exchange()
            .expectStatus().isForbidden
    }
}
```

### JWT モックの作成

```kotlin
@TestConfiguration
class TestSecurityConfig {

    @Bean
    fun jwtDecoder(): ReactiveJwtDecoder {
        return ReactiveJwtDecoder { token ->
            val claims = mapOf(
                "sub" to "12345",
                "realm_access" to mapOf("roles" to listOf("user")),
                "email" to "test@example.com",
            )
            Mono.just(
                Jwt.withTokenValue(token)
                    .header("alg", "RS256")
                    .claims { it.putAll(claims) }
                    .build()
            )
        }
    }
}
```
