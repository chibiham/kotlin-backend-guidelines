# 設定・環境管理

## 基本方針

1. **application.yml**で設定を一元管理する
2. 環境ごとの差異は**プロファイル**で管理する
3. 機密情報は**環境変数**または**外部シークレット管理サービス**で管理する
4. 設定値は**@ConfigurationProperties**で型安全にバインドする

## ファイル構成

```
src/main/resources/
├── application.yml              # 本番デフォルト設定 + プロファイル別import設定
├── application-local.yml        # ローカル開発用の上書き設定
└── application-e2e.yml          # E2Eテスト用の上書き設定
```

### 設計方針

#### 1. プロファイル戦略

| 環境 | プロファイル指定 | 設定ファイル | 環境変数 |
|------|----------------|-------------|---------|
| **本番環境** | **指定なし**（デフォルト） | `application.yml` | 必須 |
| **ローカル開発** | `local` | `application.yml` + `application-local.yml` | 不要 |
| **E2Eテスト** | `local,e2e` | `application.yml` + `application-local.yml` + `application-e2e.yml` | 不要 |

**重要:** 本番環境では`SPRING_PROFILES_ACTIVE`を指定せず、デフォルトの`application.yml`を使用する。

#### 2. application.ymlには本番相当のデフォルト値を定義

- 環境固有の値（接続先、クレデンシャル等）は環境変数で上書き
- 環境変数が必須の場合は`${ENV_VAR}`形式で記述（デフォルト値なし）
- 末尾に`spring.config.activate.on-profile`と`import`でプロファイル別設定を読み込む

#### 3. application-local.ymlとapplication-e2e.ymlで上書き

- デフォルト設定を上書きする形で必要な設定のみを記述
- `local`: ローカル開発環境用の設定（DB接続、OAuth設定など）
- `e2e`: E2Eテスト用の設定（タイムアウト短縮、リトライ最小化など）
- E2Eテストは`local,e2e`両方のプロファイルを適用して実行
- 環境変数なしで起動できるよう具体的な値を記述

#### 4. 環境別設定ファイル（dev, staging, prod等）は作成しない

- 環境ごとの差異は環境変数で管理
- 設定ファイルの肥大化を防ぎ、管理を簡素化

## application.yml

### 基本構成（本番相当のデフォルト値）

```yaml
# application.yml - 本番相当のデフォルト値
spring:
  application:
    name: my-app

  # WebFlux設定
  webflux:
    base-path: /api

  # R2DBC設定（環境変数必須）
  r2dbc:
    url: ${SPRING_R2DBC_URL}           # 必須: 環境変数で指定
    username: ${SPRING_R2DBC_USERNAME} # 必須: 環境変数で指定
    password: ${SPRING_R2DBC_PASSWORD} # 必須: 環境変数で指定
    pool:
      initialSize: 10
      maxSize: 50
      maxIdleTime: 30m

  # Flyway設定（環境変数必須）
  flyway:
    enabled: true
    url: ${SPRING_FLYWAY_URL}           # 必須: 環境変数で指定
    user: ${SPRING_FLYWAY_USER}         # 必須: 環境変数で指定
    password: ${SPRING_FLYWAY_PASSWORD} # 必須: 環境変数で指定
    locations: classpath:db/migration

  # OAuth2 Resource Server設定（環境変数必須）
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspectionUri: ${SPRING_SECURITY_OAUTH2_RESOURCESERVER_OPAQUETOKEN_INTROSPECTIONURI}
          clientId: ${SPRING_SECURITY_OAUTH2_RESOURCESERVER_OPAQUETOKEN_CLIENTID}
          clientSecret: ${SPRING_SECURITY_OAUTH2_RESOURCESERVER_OPAQUETOKEN_CLIENTSECRET}

# サーバー設定
server:
  port: 8080
  shutdown: graceful

# アクチュエーター設定
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      showDetails: whenAuthorized
      probes:
        enabled: true
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true

# ロギング設定（本番用・JSON形式）
logging:
  level:
    root: WARN
    com.example.myapp: INFO
    org.springframework.r2dbc: WARN

# アプリケーション固有の設定（環境変数必須）
app:
  cors:
    allowedOrigins: ${APP_CORS_ALLOWEDORIGINS}  # 必須: カンマ区切りで指定
    maxAge: 3600

  rateLimit:
    enabled: true
    requestsPerMinute: 1000

  externalApi:
    baseUrl: ${APP_EXTERNALAPI_BASEURL}         # 必須: 環境変数で指定
    apiKey: ${APP_EXTERNALAPI_APIKEY}           # 必須: 環境変数で指定
    timeout: 30s
    retry:
      maxAttempts: 3
      delay: 1s

---
# ローカル開発環境用の設定をインポート
spring:
  config:
    activate:
      on-profile: local
    import: application-local.yml

---
# E2Eテスト環境用の設定をインポート
spring:
  config:
    activate:
      on-profile: e2e
    import: application-e2e.yml
```

### application-local.yml（ローカル開発用の上書き設定）

**デフォルト設定を上書きする形で、ローカル開発に必要な設定を記述する。環境変数が必須の値も含めて、環境変数なしで起動できるよう具体的な値を記述する。**

**E2Eテスト実行時は`local,e2e`の両プロファイルを適用することで、この設定をベースにE2E用の調整を行う。**

```yaml
# application-local.yml - ローカル開発用の上書き設定
spring:
  # R2DBC設定（環境変数不要）
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/myapp_local
    username: postgres
    password: postgres
    pool:
      initialSize: 5
      maxSize: 10

  # Flyway設定
  flyway:
    url: jdbc:postgresql://localhost:5432/myapp_local
    user: postgres
    password: postgres

  # OAuth2 Resource Server設定
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspectionUri: http://localhost:8180/realms/local/protocol/openid-connect/token/introspect
          clientId: my-app-local
          clientSecret: local-secret

# ロギング設定（開発用・JSON形式）
logging:
  level:
    root: INFO
    com.example.myapp: DEBUG
    org.springframework.r2dbc: DEBUG

# アプリケーション固有の設定
app:
  cors:
    allowedOrigins: http://localhost:3000,http://localhost:5173

  rateLimit:
    enabled: false # ローカルでは無効化
    requestsPerMinute: 100

  externalApi:
    baseUrl: http://localhost:8081 # ローカルのモックサーバー
    apiKey: local-api-key
```

### application-e2e.yml（E2Eテスト用の上書き設定）

**`local,e2e`の両プロファイルを適用することを前提とした設定ファイル。`application-local.yml`の設定をベースに、E2Eテスト特有の調整（タイムアウト短縮、リトライ最小化など）のみを上書きする。**

**DB接続情報やOAuth設定などは`application-local.yml`から継承するため、E2E特有の設定のみを記述する。**

```yaml
# application-e2e.yml - E2Eテスト用の上書き設定
# プロファイル適用: local,e2e（localの設定をベースにE2E用に調整）

# R2DBC接続プール設定（E2E用に調整）
spring:
  r2dbc:
    pool:
      initialSize: 3
      maxSize: 5
      maxIdleTime: 10m

# ロギング設定（テスト用に調整・JSON形式）
logging:
  level:
    root: WARN
    com.example.myapp: INFO
    org.springframework.r2dbc: WARN

# アプリケーション固有の設定（E2E用に調整）
app:
  externalApi:
    timeout: 5s       # タイムアウトを短く設定
    retry:
      maxAttempts: 1  # リトライを最小化
      delay: 100ms    # 遅延を短く設定
```

## @ConfigurationProperties

### 設定クラスの定義

```kotlin
@ConfigurationProperties(prefix = "app")
data class AppProperties(
    val cors: CorsProperties,
    val rateLimit: RateLimitProperties,
    val externalApi: ExternalApiProperties,
)

data class CorsProperties(
    val allowedOrigins: List<String> = emptyList(),
    val maxAge: Long = 3600,
)

data class RateLimitProperties(
    val enabled: Boolean = true,
    val requestsPerMinute: Int = 100,
)

data class ExternalApiProperties(
    val baseUrl: String,
    val timeout: Duration = Duration.ofSeconds(30),
    val retry: RetryProperties = RetryProperties(),
) {
    data class RetryProperties(
        val maxAttempts: Int = 3,
        val delay: Duration = Duration.ofSeconds(1),
    )
}
```

### 有効化

```kotlin
@Configuration
@ConfigurationPropertiesScan("com.example.myapp.config")
class PropertiesConfig
```

または個別に指定：

```kotlin
@Configuration
@EnableConfigurationProperties(AppProperties::class)
class PropertiesConfig
```

### 使用例

```kotlin
@Service
class ExternalApiService(
    private val appProperties: AppProperties,
    private val webClient: WebClient,
) {
    private val externalApiConfig = appProperties.externalApi

    suspend fun fetchData(): Data {
        return webClient.get()
            .uri(externalApiConfig.baseUrl + "/data")
            .retrieve()
            .awaitBody()
    }
}

@Configuration
class WebClientConfig(private val appProperties: AppProperties) {

    @Bean
    fun webClient(): WebClient {
        val timeout = appProperties.externalApi.timeout

        return WebClient.builder()
            .baseUrl(appProperties.externalApi.baseUrl)
            .clientConnector(
                ReactorClientHttpConnector(
                    HttpClient.create()
                        .responseTimeout(timeout)
                )
            )
            .build()
    }
}
```

## バリデーション

### 設定値のバリデーション

```kotlin
@ConfigurationProperties(prefix = "app.externalApi")
@Validated
data class ExternalApiProperties(
    @field:NotBlank(message = "Base URL is required")
    val baseUrl: String,

    @field:NotNull
    @field:DurationMin(seconds = 1)
    @field:DurationMax(minutes = 5)
    val timeout: Duration = Duration.ofSeconds(30),

    @field:Valid
    val retry: RetryProperties = RetryProperties(),
) {
    @Validated
    data class RetryProperties(
        @field:Min(1)
        @field:Max(10)
        val maxAttempts: Int = 3,

        @field:DurationMin(millis = 100)
        val delay: Duration = Duration.ofSeconds(1),
    )
}
```

### カスタムバリデーション

```kotlin
@ConfigurationProperties(prefix = "app.database")
@Validated
data class DatabaseProperties(
    val pool: PoolProperties,
) {
    init {
        require(pool.maxSize >= pool.initialSize) {
            "maxSize must be >= initialSize"
        }
    }

    data class PoolProperties(
        val initialSize: Int = 5,
        val maxSize: Int = 20,
    )
}
```

## 環境変数

### 命名規則

**独自設定の YAML プロパティ命名規則:**

1. **ハイフンを使用しない** - 名前空間の区切りには`.`（ドット）を使用
2. **変数名は camelCase** - プロパティ名は camelCase で記述
3. **環境変数は UPPER_SNAKE_CASE** - 環境変数は大文字アンダースコア区切り

この規則により、YAML から環境変数への変換ルールが明確になり、ハイフン除去による曖昧さを回避できます。

**変換ルール:**

```
YAML: xxx.yyy.variableName
  ↓
環境変数: XXX_YYY_VARIABLENAME
```

**適用範囲:** この規則は`app.*`以下の独自設定にのみ適用し、Spring やライブラリの設定には適用しません。

#### 独自設定の例

```yaml
# ✅ 推奨：camelCase、ハイフンなし
app:
  externalApi:
    baseUrl: ${APP_EXTERNALAPI_BASEURL:https://api.example.com}
    apiKey: ${APP_EXTERNALAPI_APIKEY}
    timeout: ${APP_EXTERNALAPI_TIMEOUT:30s}
  rateLimit:
    enabled: ${APP_RATELIMIT_ENABLED:true}
    requestsPerMinute: ${APP_RATELIMIT_REQUESTSPERMINUTE:100}
  cors:
    allowedOrigins: ${APP_CORS_ALLOWEDORIGINS:http://localhost:3000}
    maxAge: ${APP_CORS_MAXAGE:3600}

# ❌ 非推奨：ハイフン混在（変換時に曖昧になる）
app:
  external-api:          # NG: ハイフン使用
    base-url: "..."      # NG: ハイフン使用
  rate-limit:            # NG: ハイフン使用
    requests-per-minute: # NG: ハイフン使用
```

#### Spring/ライブラリ設定の例

```yaml
# Spring組み込み設定はそのまま（ハイフンあり）
spring:
  r2dbc:
    url: ${SPRING_R2DBC_URL}
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: ${SPRING_SECURITY_OAUTH2_RESOURCESERVER_OPAQUETOKEN_INTROSPECTION_URI}
          client-id: ${SPRING_SECURITY_OAUTH2_RESOURCESERVER_OPAQUETOKEN_CLIENT_ID}
```

#### 命名規則の比較

| 設定種別     | YAML 記法                                                             | 環境変数                                                              |
| ------------ | --------------------------------------------------------------------- | --------------------------------------------------------------------- |
| **独自設定** | `app.externalApi.baseUrl`                                             | `APP_EXTERNALAPI_BASEURL`                                             |
| **独自設定** | `app.rateLimit.requestsPerMinute`                                     | `APP_RATELIMIT_REQUESTSPERMINUTE`                                     |
| Spring 設定  | `spring.r2dbc.url`                                                    | `SPRING_R2DBC_URL`                                                    |
| Spring 設定  | `spring.security.oauth2.resourceserver.opaquetoken.introspection-uri` | `SPRING_SECURITY_OAUTH2_RESOURCESERVER_OPAQUETOKEN_INTROSPECTION_URI` |

#### ハイフン使用時の問題例

```yaml
# ❌ ハイフンは環境変数変換時に削除されるため曖昧
app:
  external-api:
    base-url: "..." # → APP_EXTERNALAPI_BASEURL

  externalapi: # ← 上記と区別できない！
    baseurl: "..." # → APP_EXTERNALAPI_BASEURL（同じ環境変数名）
```

### 環境変数の使い分け

#### 必須環境変数（デフォルト値なし）

接続先やクレデンシャルなど、環境ごとに必ず異なる値は環境変数必須とする。

```yaml
# ✅ 推奨：環境変数必須（デフォルト値なし）
spring:
  r2dbc:
    url: ${SPRING_R2DBC_URL} # 必須
    username: ${SPRING_R2DBC_USERNAME} # 必須
    password: ${SPRING_R2DBC_PASSWORD} # 必須

app:
  externalApi:
    baseUrl: ${APP_EXTERNALAPI_BASEURL} # 必須
    apiKey: ${APP_EXTERNALAPI_APIKEY} # 必須
```

#### デフォルト値あり（任意の環境変数）

環境によって変更する可能性があるが、デフォルト値で問題ない設定。

```yaml
# ✅ 任意：デフォルト値あり
server:
  port: ${SERVER_PORT:8080} # デフォルトは8080

app:
  rateLimit:
    requestsPerMinute: ${APP_RATELIMIT_REQUESTSPERMINUTE:1000} # デフォルトは1000
```

#### 固定値（環境変数なし）

どの環境でも同じ値を使う設定は、環境変数を使わず直接記述。

```yaml
# ✅ 固定値：環境変数不要
server:
  shutdown: graceful

spring:
  flyway:
    locations: classpath:db/migration

app:
  cors:
    maxAge: 3600
```

## 機密情報の管理

### 環境変数での管理

```yaml
# 機密情報は環境変数から取得
spring:
  r2dbc:
    password: ${DB_PASSWORD}

app:
  externalApi:
    apiKey: ${APP_EXTERNALAPI_APIKEY}
    apiSecret: ${APP_EXTERNALAPI_APISECRET}
```

### Kubernetes Secrets

```yaml
# kubernetes/secrets.yml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
type: Opaque
stringData:
  DB_PASSWORD: "secure-password"
  EXTERNAL_API_KEY: "api-key-value"
```

```yaml
# kubernetes/deployment.yml
spec:
  containers:
    - name: myapp
      envFrom:
        - secretRef:
            name: myapp-secrets
```

### AWS Secrets Manager / Parameter Store

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.awspring.cloud:spring-cloud-aws-starter-secrets-manager:3.1.0")
}
```

```yaml
# application.yml
spring:
  config:
    import: aws-secretsmanager:/secrets/myapp/
# AWS Secrets Managerの値が自動的にプロパティにバインドされる
```

### HashiCorp Vault

```kotlin
// build.gradle.kts
dependencies {
    implementation("org.springframework.cloud:spring-cloud-starter-vault-config")
}
```

```yaml
# bootstrap.yml
spring:
  cloud:
    vault:
      uri: ${VAULT_ADDR:http://localhost:8200}
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        backend: secret
        default-context: myapp
```

## プロファイル

### プロファイルの有効化

プロファイル戦略は以下の通り：

| 環境 | プロファイル指定 | 起動方法 |
|------|----------------|---------|
| 本番環境 | **指定なし** | `java -jar myapp.jar` |
| ローカル開発 | `local` | `java -jar myapp.jar --spring.profiles.active=local` |
| E2Eテスト | `local,e2e` | `java -jar myapp.jar --spring.profiles.active=local,e2e` |

#### 本番環境

**プロファイルを指定しない**ことで、`application.yml`のデフォルト設定が使用される。

```bash
# 本番環境: プロファイル指定なし
java -jar myapp.jar

# 環境変数も設定しない（設定すると意図しないプロファイルが有効化される可能性がある）
# SPRING_PROFILES_ACTIVE は設定しない
```

**注意:** `SPRING_PROFILES_ACTIVE=default`のような明示的な指定は不要。プロファイル未指定が本番環境のデフォルト動作。

#### ローカル開発環境

```bash
# ローカル開発環境
export SPRING_PROFILES_ACTIVE=local
java -jar myapp.jar

# または起動時に指定
java -jar myapp.jar --spring.profiles.active=local

# Gradleでの起動
./gradlew bootRun --args='--spring.profiles.active=local'
```

#### E2Eテスト環境

```bash
# E2Eテスト環境（localとe2eの両方を適用）
export SPRING_PROFILES_ACTIVE=local,e2e
java -jar myapp.jar

# または起動時に指定
java -jar myapp.jar --spring.profiles.active=local,e2e

# Gradleでの起動
./gradlew bootRun --args='--spring.profiles.active=local,e2e'
```

### プロファイルによる Bean 切り替え

```kotlin
@Configuration
class DataSourceConfig {

    @Bean
    @Profile("local")
    fun localDataSource(): ConnectionFactory {
        // ローカル用の設定
        return ConnectionFactories.get("r2dbc:h2:mem:///testdb")
    }

    @Bean
    @Profile("!local")
    fun productionDataSource(properties: R2dbcProperties): ConnectionFactory {
        // 本番用の設定
        return ConnectionFactories.get(properties.url)
    }
}

@Component
@Profile("local", "e2e")
class MockExternalService : ExternalService {
    override suspend fun call(): Response = Response.mock()
}

@Component
@Profile("!local & !e2e")
class RealExternalService(
    private val webClient: WebClient,
) : ExternalService {
    override suspend fun call(): Response {
        return webClient.get().retrieve().awaitBody()
    }
}
```

### 条件付き Bean 登録

```kotlin
@Configuration
class FeatureConfig {

    @Bean
    @ConditionalOnProperty(
        prefix = "app.feature",
        name = ["newAlgorithm"],
        havingValue = "true",
    )
    fun newAlgorithm(): Algorithm = NewAlgorithm()

    @Bean
    @ConditionalOnProperty(
        prefix = "app.feature",
        name = ["newAlgorithm"],
        havingValue = "false",
        matchIfMissing = true,
    )
    fun legacyAlgorithm(): Algorithm = LegacyAlgorithm()
}
```

## ロギング設定

ロギングに関する詳細なガイドラインは、[10-logging.md](./10-logging.md)を参照してください。

- JSON形式でのログ出力設定
- New Relic TraceIdとの連携
- CoroutineでのMDC伝搬
- Kotlin特有のロギングベストプラクティス

## ヘルスチェック

### カスタムヘルスインジケーター

```kotlin
@Component
class ExternalApiHealthIndicator(
    private val webClient: WebClient,
    private val appProperties: AppProperties,
) : ReactiveHealthIndicator {

    override fun health(): Mono<Health> {
        return webClient.get()
            .uri(appProperties.externalApi.baseUrl + "/health")
            .retrieve()
            .toBodilessEntity()
            .map { Health.up().build() }
            .onErrorResume { ex ->
                Mono.just(
                    Health.down()
                        .withDetail("error", ex.message)
                        .build()
                )
            }
            .timeout(Duration.ofSeconds(5))
    }
}

@Component
class DatabaseHealthIndicator(
    private val connectionFactory: ConnectionFactory,
) : ReactiveHealthIndicator {

    override fun health(): Mono<Health> {
        return Mono.from(connectionFactory.create())
            .flatMap { connection ->
                Mono.from(connection.createStatement("SELECT 1").execute())
                    .doFinally { Mono.from(connection.close()).subscribe() }
            }
            .map { Health.up().build() }
            .onErrorResume { ex ->
                Mono.just(
                    Health.down()
                        .withDetail("error", ex.message)
                        .build()
                )
            }
    }
}
```

### ヘルスチェックエンドポイント

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info

  endpoint:
    health:
      show-details: when_authorized
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState,db,externalApi
```

```
GET /actuator/health/liveness  -> Kubernetes liveness probe
GET /actuator/health/readiness -> Kubernetes readiness probe
```

## 設定の確認

### 起動時のログ出力

```kotlin
@Component
class ConfigurationLogger(
    private val appProperties: AppProperties,
    private val environment: Environment,
) {
    private val logger = KotlinLogging.logger {}

    @EventListener(ApplicationReadyEvent::class)
    fun logConfiguration() {
        logger.info { "=== Application Configuration ===" }
        logger.info { "Active profiles: ${environment.activeProfiles.joinToString()}" }
        logger.info { "External API URL: ${appProperties.externalApi.baseUrl}" }
        logger.info { "Rate limit: ${appProperties.rateLimit.requestsPerMinute} req/min" }
        logger.info { "CORS origins: ${appProperties.cors.allowedOrigins}" }
        logger.info { "=================================" }
    }
}
```

### 設定確認エンドポイント（開発用）

```kotlin
@RestController
@RequestMapping("/admin/config")
@Profile("local", "e2e")  // 本番では無効
class ConfigController(
    private val appProperties: AppProperties,
) {
    @GetMapping
    fun getConfig(): Map<String, Any> {
        return mapOf(
            "externalApi" to mapOf(
                "baseUrl" to appProperties.externalApi.baseUrl,
                "timeout" to appProperties.externalApi.timeout.toString(),
            ),
            "rateLimit" to mapOf(
                "enabled" to appProperties.rateLimit.enabled,
                "requestsPerMinute" to appProperties.rateLimit.requestsPerMinute,
            ),
        )
    }
}
```

## Docker / Kubernetes

### Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY build/libs/*.jar app.jar

# 非rootユーザーで実行
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Kubernetes ConfigMap

```yaml
# kubernetes/configmap.yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
data:
  # 本番環境ではSPRING_PROFILES_ACTIVEを設定しない（デフォルト使用）
  # SPRING_PROFILES_ACTIVE: ""  # 設定しない
  SERVER_PORT: "8080"
  APP_CORS_ALLOWEDORIGINS: "https://app.example.com"
  APP_RATELIMIT_REQUESTSPERMINUTE: "1000"
```

### Kubernetes Deployment

```yaml
# kubernetes/deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: myapp-config
            - secretRef:
                name: myapp-secrets
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
```

## ローカル開発環境のセットアップ

### READMEへの記載

```markdown
## 環境別の起動方法

### ローカル開発環境

`local`プロファイルを指定して起動してください。環境変数の設定は不要です。

```bash
# Gradleでの起動
./gradlew bootRun --args='--spring.profiles.active=local'

# またはJARファイルから起動
java -jar build/libs/myapp.jar --spring.profiles.active=local
```

データベースやOAuth設定などは`application-local.yml`に記述されています。

### E2Eテスト環境

`local,e2e`の両プロファイルを指定して実行してください。

```bash
./gradlew bootRun --args='--spring.profiles.active=local,e2e'
```

### 本番環境

**プロファイルを指定せず**に起動してください。環境変数で設定を上書きします。

```bash
# プロファイル指定なし
java -jar myapp.jar
```

必須の環境変数：
- `SPRING_R2DBC_URL`
- `SPRING_R2DBC_USERNAME`
- `SPRING_R2DBC_PASSWORD`
- `SPRING_SECURITY_OAUTH2_RESOURCESERVER_OPAQUETOKEN_INTROSPECTIONURI`
- `SPRING_SECURITY_OAUTH2_RESOURCESERVER_OPAQUETOKEN_CLIENTID`
- `SPRING_SECURITY_OAUTH2_RESOURCESERVER_OPAQUETOKEN_CLIENTSECRET`
- `APP_CORS_ALLOWEDORIGINS`
- `APP_EXTERNALAPI_BASEURL`
- `APP_EXTERNALAPI_APIKEY`
```

## チェックリスト

### 本番環境デプロイ前の確認

#### プロファイル設定
- [ ] **`SPRING_PROFILES_ACTIVE`環境変数を設定していない**（プロファイル指定なし）
- [ ] `application-local.yml`と`application-e2e.yml`が本番環境にデプロイされていない
- [ ] デバッグ用エンドポイントが無効化されている（`@Profile("local", "e2e")`）

#### セキュリティ
- [ ] 機密情報がコードやYAMLにハードコードされていない
- [ ] 必須環境変数がすべて設定されている
- [ ] CORSの許可オリジンが本番URLのみ

#### 設定
- [ ] `application.yml`のデフォルト値が本番相当になっている
- [ ] ログレベルが適切（本番はWARN/INFO）
- [ ] ログがJSON形式で出力されている（LogstashEncoder）
- [ ] ヘルスチェックエンドポイントが設定されている
- [ ] データベース接続プールのサイズが適切
- [ ] タイムアウト値が設定されている
- [ ] 適切なリソース制限（CPU、メモリ）が設定されている
