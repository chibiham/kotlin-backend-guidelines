# ロギング

## 基本方針

1. **JSON 形式で統一** - すべての環境で LogstashEncoder を使用
2. **構造化ログ** - 検索・分析しやすい形式で出力
3. **TraceId で追跡** - リクエスト全体を通してログを紐付ける
4. **適切なログレベル** - 環境ごとに最適なレベルを設定
5. **MDC でコンテキスト情報を付与** - TraceId などを自動付与

## Logback 設定

### 依存関係の追加

```kotlin
// build.gradle.kts
dependencies {
    implementation("net.logstash.logback:logstash-logback-encoder:7.4")
    implementation("io.github.microutils:kotlin-logging-jvm:3.0.5")
}
```

### Logback 設定ファイル

**全環境で JSON 形式のログを出力します。**

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- JSON形式のAppender（全環境共通） -->
    <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- MDCフィールド -->
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>

            <!-- タイムスタンプ形式 -->
            <timestampPattern>yyyy-MM-dd'T'HH:mm:ss.SSSZZ</timestampPattern>

            <!-- カスタムフィールド -->
            <customFields>{"application":"my-app","env":"${ENVIRONMENT:-local}"}</customFields>
        </encoder>
    </appender>

    <!-- 全プロファイルで共通のAppenderを使用 -->
    <root level="INFO">
        <appender-ref ref="JSON_CONSOLE"/>
    </root>

    <!-- ログレベルはapplication.ymlで制御 -->
</configuration>
```

### ログレベルの制御

ログレベルは`application.yml`および各プロファイル設定ファイルで環境ごとに制御します。

```yaml
# application.yml（本番用）
logging:
  level:
    root: WARN
    com.example.myapp: INFO
    org.springframework.r2dbc: WARN
    org.springframework.security: WARN

---
# application-local.yml（開発用）
logging:
  level:
    root: INFO
    com.example.myapp: DEBUG
    org.springframework.r2dbc: DEBUG
    org.springframework.security: DEBUG

---
# application-e2e.yml（E2Eテスト用）
logging:
  level:
    root: WARN
    com.example.myapp: INFO
    org.springframework.r2dbc: WARN
```

## ログ出力の基本

### Kotlin Logging の使用（推奨）

Kotlin では SLF4J を直接使わず、**kotlin-logging（KotlinLogging）** を使用します。

**依存関係の追加:**

```kotlin
// build.gradle.kts
dependencies {
    implementation("io.github.microutils:kotlin-logging-jvm:3.0.5")
    implementation("ch.qos.logback:logback-classic:1.4.14") // SLF4J実装
}
```

**基本的な使い方:**

```kotlin
import mu.KotlinLogging

// トップレベルでloggerを定義（companion object不要）
private val logger = KotlinLogging.logger {}

@Service
class UserService(
    private val userRepository: UserRepository,
) {
    suspend fun findById(id: UserId): User? {
        // ラムダ構文で遅延評価
        logger.info { "Finding user: $id" }

        val user = userRepository.findById(id.value)?.toDomain()

        if (user == null) {
            logger.warn { "User not found: $id" }
        }

        return user
    }

    suspend fun create(request: CreateUserRequest): User {
        logger.info { "Creating user: email=${request.email}" }

        try {
            val user = userRepository.save(request.toEntity())
            logger.info { "User created successfully: id=${user.id}" }
            return user.toDomain()
        } catch (e: DuplicateKeyException) {
            logger.error(e) { "Failed to create user: duplicate email=${request.email}" }
            throw ApplicationException.BadRequest.DuplicateEmail(request.email)
        }
    }
}
```

### KotlinLogging を推奨する理由

#### 1. 遅延評価による性能向上

```kotlin
// ❌ SLF4J: ログレベルが無効でも文字列連結が実行される
logger.debug("User data: ${expensiveOperation()}")

// ✅ KotlinLogging: DEBUGレベルが無効なら expensiveOperation() は呼ばれない
logger.debug { "User data: ${expensiveOperation()}" }
```

#### 2. Kotlin イディオムに沿った記述

```kotlin
// ❌ SLF4J: Javaスタイル（冗長）
class UserService {
    companion object {
        private val logger = LoggerFactory.getLogger(UserService::class.java)
    }

    fun findUser(id: String) {
        if (logger.isDebugEnabled) {
            logger.debug("Finding user: $id")
        }
    }
}

// ✅ KotlinLogging: Kotlinスタイル（簡潔）
private val logger = KotlinLogging.logger {}

class UserService {
    fun findUser(id: String) {
        logger.debug { "Finding user: $id" }
    }
}
```

#### 3. 例外処理が簡潔

```kotlin
// ❌ SLF4J: 例外を第1引数に渡す
catch (e: Exception) {
    logger.error("Failed to process", e)
}

// ✅ KotlinLogging: 例外を引数、メッセージをラムダで
catch (e: Exception) {
    logger.error(e) { "Failed to process: userId=$userId" }
}
```

### ログレベルの使い分け

| レベル    | 用途                           | 例                                                |
| --------- | ------------------------------ | ------------------------------------------------- |
| **ERROR** | 予期しないエラー、システム障害 | 外部 API 障害、DB 接続エラー、予期しない例外      |
| **WARN**  | 注意が必要な状況、潜在的な問題 | リトライ発生、非推奨 API 使用、ビジネスルール違反 |
| **INFO**  | 重要なビジネスイベント         | ユーザー作成、注文完了、重要な状態変更            |
| **DEBUG** | 詳細なデバッグ情報（開発用）   | クエリパラメータ、中間結果、内部状態              |

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val paymentGateway: PaymentGateway,
) {
    suspend fun processOrder(orderId: OrderId) {
        logger.info { "Processing order: $orderId" }  // ビジネスイベント

        val order = orderRepository.findById(orderId.value)
            ?: throw ApplicationException.NotFound.Order(orderId)

        logger.debug { "Order details: status=${order.status}, amount=${order.amount}" }  // デバッグ情報

        try {
            val result = paymentGateway.charge(order.amount)
            logger.info { "Payment completed: orderId=$orderId, transactionId=${result.transactionId}" }
        } catch (e: PaymentException) {
            logger.error(e) { "Payment failed: orderId=$orderId, reason=${e.reason}" }  // エラー
            throw e
        }
    }
}
```

## New Relic とログの連携

### 調査フロー例: 「注文処理が失敗した」問題のデバッグ

**1. New Relic でトレースを確認** - どこで時間がかかっているかを特定

```
TraceId: abc123
├─ Controller.createOrder - 2500ms
│  ├─ OrderService.processOrder - 2400ms
│  │  ├─ Database.query - 50ms
│  │  ├─ PaymentGateway.charge - 2300ms ← ここが遅い！
│  │  └─ Database.update - 20ms
```

**わかること:** PaymentGateway で 2.3 秒かかっている

**わからないこと:** なぜ失敗したのか、どんなエラーか

**2. TraceId でログを検索** - ビジネスロジックの詳細を確認

```bash
# CloudWatch / Datadog / Elasticsearchで検索
filter traceId="abc123"
```

```json
{"traceId":"abc123","level":"INFO","message":"Processing order: 12345"}
{"traceId":"abc123","level":"INFO","message":"Order amount: 25000"}
{"traceId":"abc123","level":"WARN","message":"High-value order detected: 12345"}
{"traceId":"abc123","level":"ERROR","message":"Payment failed: Card declined - insufficient funds"}
```

**わかること:** 注文 ID、金額、失敗理由（カード残高不足）

**3. 両方を組み合わせて完全な情報を得る**

- New Relic: 処理フローと性能ボトルネック
- ログ: ビジネスロジックの詳細とエラー原因

### New Relic の仕組みとログ連携の必要性

New Relic の一部のプランでは、ログを直接 New Relic に送信して統合表示できます。その場合でも、TraceId がログに含まれていることで自動的に紐付けが行われます。

New Relic は**Java Agent**として動作し、バイトコードを実行時に書き換えて自動的にトレース情報を収集します。

```bash
# アプリケーション起動時
java -javaagent:/path/to/newrelic.jar -jar myapp.jar
```

**New Relic が自動収集する情報:**

- 処理フロー（どのメソッドが呼ばれたか）
- 各処理の実行時間
- DB クエリ、HTTP リクエストの詳細
- TraceId と SpanId（階層構造）

**New Relic が収集しない情報:**

- **アプリケーションログの内容**（`logger.info { }` などの出力）
- ビジネスロジックの詳細
- デバッグ用の変数値
- エラーの詳細なコンテキスト

そのため、**New Relic とアプリケーションログは補完関係**にあります：

| 用途               | New Relic                 | アプリケーションログ       |
| ------------------ | ------------------------- | -------------------------- |
| **どこが遅いか**   | ✅ 処理時間を可視化       | ❌                         |
| **何が呼ばれたか** | ✅ メソッド呼び出しを記録 | ❌                         |
| **なぜ失敗したか** | ❌                        | ✅ エラーメッセージと詳細  |
| **ビジネスデータ** | ❌                        | ✅ 注文 ID、金額などの詳細 |

**TraceId をログに含めることで、New Relic のトレース画面とログを相互に行き来して調査できます。**

**TraceId と SpanId の違い:**

| 項目               | TraceId                | SpanId                           |
| ------------------ | ---------------------- | -------------------------------- |
| **粒度**           | リクエスト全体         | 個々の処理単位                   |
| **数**             | リクエストごとに 1 つ  | リクエスト内で複数               |
| **用途**           | 関連する処理をまとめる | 処理の内訳を追跡                 |
| **ログへの含め方** | Filter で設定          | 各処理で動的に取得（通常は不要） |

SpanId の階層構造は New Relic が自動的に管理するため、アプリケーションログに含める必要はありません。

### traceId の出力と New Relic が無効な場合のフォールバック

**重要:** TraceId のみを MDC に設定します。SpanId は処理ごとに変わるため、Filter レベルで設定しても意味がありません。

**注意:** New Relic エージェントが有効でない場合（ローカル開発環境など）、`traceId`は空文字列になる可能性があります。その場合は独自のトレース ID を生成することを検討してください。

```kotlin
import com.newrelic.api.agent.NewRelic
import org.slf4j.MDC
import org.springframework.core.Ordered
import org.springframework.core.annotation.Order
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import reactor.core.publisher.Mono
import java.util.UUID

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
class TraceIdMdcFilter : WebFilter {

    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        // New Relicが無効な場合のフォールバック
        val traceId = try {
            NewRelic.getAgent().traceMetadata.traceId.takeIf { it.isNotBlank() }
        } catch (e: Exception) {
            null
        } ?: UUID.randomUUID().toString()

        MDC.put("traceId", traceId)

        return chain.filter(exchange)
    }
}
```

## MDC 連携

### 依存関係の追加

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.newrelic.agent.java:newrelic-api:8.8.0")
    implementation("io.micrometer:context-propagation:1.0.6")
}
```

### コンテキスト伝播の設定

WebFlux はイベント駆動・非同期処理のため、ThreadLocal ベースの MDC は異なるスレッド間で自動的に伝播されません。**Micrometer Context Propagation**を使用することで、MDC が Reactor Context に自動的に伝播されます。

#### Spring Boot 3 での設定（必須）

Spring Boot 3 では、以下の設定が**必須**です。この設定がないと、MDC の値は Reactor Context に伝播されません：

```yaml
# application.yml
spring:
  reactor:
    context-propagation: auto
```

この設定により、Spring Boot が自動的に`Hooks.enableAutomaticContextPropagation()`を呼び出し、MDC の値が Reactor Context に伝播されるようになります。

**重要:** この設定は**デフォルトで無効**なため、必ず明示的に設定する必要があります

#### WebFilter での MDC 設定

```kotlin
import com.newrelic.api.agent.NewRelic
import org.slf4j.MDC
import org.springframework.core.Ordered
import org.springframework.core.annotation.Order
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import reactor.core.publisher.Mono

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
class TraceIdMdcFilter : WebFilter {

    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        // New RelicからTraceIdのみを取得
        // TraceIdはリクエスト全体で1つなので、Filterで設定する意味がある
        val traceId = NewRelic.getAgent().traceMetadata.traceId

        // MDCにTraceIdのみを設定
        // Micrometer Context Propagationが以下を自動処理:
        // - Reactor Contextへの伝播
        // - スレッド切り替え時の復元
        // - リクエスト完了時のクリーンアップ
        MDC.put("traceId", traceId)

        return chain.filter(exchange)
    }
}
```

**MDC.remove()は不要**

`Hooks.enableAutomaticContextPropagation()`により、Micrometer Context Propagation が自動的に ThreadLocal のライフサイクルを管理します：

1. **伝播**: MDC の値が Reactor Context に自動的にコピーされる
2. **復元**: スレッド切り替え時に、Reactor Context から MDC に復元される
3. **クリーンアップ**: リクエスト処理完了時に、ThreadLocal が自動的にクリアされる

したがって、`doFinally { MDC.remove("traceId") }`のような明示的なクリーンアップは不要です。これにより、ThreadLocal 汚染の心配なく、シンプルなコードを保つことができます。

### Micrometer Context Propagation の仕組み

**Micrometer Context Propagation**は、ThreadLocal（MDC 等）と Reactor Context の橋渡しをする仕組みです。

#### 従来の課題

WebFlux はイベント駆動・非同期処理のため、ThreadLocal ベースの MDC は以下の問題がありました：

1. **スレッド切り替え時にデータが失われる**: `subscribeOn()`や`publishOn()`でスレッドが変わると MDC が失われる
2. **ThreadLocal 汚染のリスク**: スレッドプールは複数リクエストで共有されるため、前のリクエストの MDC 値が残る可能性

#### Micrometer Context Propagation による解決

`Hooks.enableAutomaticContextPropagation()`を有効化すると：

1. **MDC → Reactor Context**: MDC に設定された値が自動的に Reactor Context にコピーされる
2. **Reactor Context → MDC**: スレッドが切り替わる際、Reactor Context の値が MDC に復元される
3. **自動クリーンアップ**: リクエスト処理完了時に、ThreadLocal が自動的にクリアされる
4. **リクエストスコープの維持**: Reactor Context はリクエストスコープで管理されるため、リクエスト間の混在が防止される

**つまり、通常の MDC 設定がそのまま使え、ThreadLocal 汚染の心配もない**ようになります。

#### ThreadLocalAccessor の登録

Micrometer Context Propagation は、内部的に以下のような`ThreadLocalAccessor`を登録しています：

```kotlin
// 内部実装（簡略化）
ContextRegistry.getInstance().registerThreadLocalAccessor(
    "traceId",
    { MDC.get("traceId") },           // getValue: 読み取り
    { value -> MDC.put("traceId", value) }, // setValue: 設定
    { MDC.remove("traceId") }         // reset: クリーンアップ
)
```

このため、Reactor Context のライフサイクルに合わせて、ThreadLocal も自動的に管理されます。

#### 使用例（アプリケーションコード）

```kotlin
import mu.KotlinLogging
import org.springframework.stereotype.Service

private val logger = KotlinLogging.logger {}

@Service
class UserService(
    private val userRepository: UserRepository,
) {
    suspend fun findById(id: UserId): User? {
        // 特別な設定不要。通常通りログ出力するだけでtraceIdが含まれる
        logger.info { "Finding user: $id" }
        return userRepository.findById(id.value)?.toDomain()
    }

    suspend fun create(request: CreateUserRequest): User {
        logger.info { "Creating user: ${request.email}" }
        val user = User.create(request)
        return userRepository.save(user.toEntity()).toDomain()
    }
}
```

### ログ出力例

```json
{
  "@timestamp": "2024-01-15T10:30:45.123+09:00",
  "level": "INFO",
  "logger": "com.example.myapp.application.UserService",
  "message": "Finding user: 123",
  "traceId": "abc123def456",
  "application": "my-app"
}
```

**ログに含まれる情報:**

- `@timestamp`: ログ出力日時
- `level`: ログレベル
- `logger`: ロガー名（クラス名）
- `message`: ログメッセージ
- `traceId`: New Relic とログを紐付けるための ID
- `application`: アプリケーション名

## カスタム MDC フィールドの追加例

### UserId の追加

認証済みユーザーの ID も MDC に追加し、ログに含めます。Micrometer Context Propagation により、MDC の値は自動的に Reactor Context に伝播されます。

#### WebFilter で MDC に userId を追加

```kotlin
import org.slf4j.MDC
import org.springframework.core.Ordered
import org.springframework.core.annotation.Order
import org.springframework.security.core.Authentication
import org.springframework.security.oauth2.server.resource.authentication.BearerTokenAuthentication
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import reactor.core.publisher.Mono

@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 1)  // TraceIdMdcFilterの次に実行
class UserIdMdcFilter : WebFilter {

    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        return exchange.getPrincipal<Authentication>()
            .flatMap { authentication ->
                // OAuth 2.0のトークン情報からuserIdを取得
                val userId = when (authentication) {
                    is BearerTokenAuthentication -> {
                        authentication.tokenAttributes["sub"]?.toString()
                    }
                    else -> null
                }

                // MDCにuserIdを設定
                // Micrometer Context Propagationにより、自動的にReactor Contextに伝播され、
                // リクエスト完了時に自動クリーンアップされる
                userId?.let { MDC.put("userId", it) }

                chain.filter(exchange)
            }
            .switchIfEmpty(chain.filter(exchange))
    }
}
```

## HTTP リクエスト/レスポンスのログ出力

### デフォルトの動作

Spring WebFlux では**デフォルトではリクエスト/レスポンスの詳細ログは出力されません**。

org.springframework.web.server.adapter.HttpWebHandlerAdapter のログレベルを DEBUG に設定すると、基本的なリクエスト情報が出力されます。

```yaml
# Spring WebFluxのログレベルをDEBUGにした場合
logging:
  level:
    org.springframework.web.server.adapter.HttpWebHandlerAdapter: DEBUG
```

**出力される内容（最小限・JSON 形式）:**

```json
{
  "level": "DEBUG",
  "logger": "org.springframework.web.server.adapter.HttpWebHandlerAdapter",
  "message": "HTTP GET \"/api/users/123\"",
  "traceId": "abc123"
}
```

しかし、詳細なヘッダー情報やボディ、duration は出力されません。また、ログレベルによる柔軟な制御もできないため、カスタム実装による WebFilter でのカスタムログ出力を導入します。

### 独自 WebFilter でのログ出力

#### HttpLoggingFilter の実装

```kotlin
package com.example.myapp.filter

import mu.KotlinLogging
import net.logstash.logback.argument.StructuredArguments.kv
import org.springframework.core.Ordered
import org.springframework.core.annotation.Order
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import reactor.core.publisher.Mono

private val logger = KotlinLogging.logger {}

@Component
@Order(Ordered.HIGHEST_PRECEDENCE + 10)
class HttpLoggingFilter : WebFilter {

    private val sensitiveHeaders = setOf("authorization", "cookie", "x-api-key")

    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        val startTime = System.currentTimeMillis()

        // リクエストログ出力
        logRequest(exchange)

        // レスポンスログ出力
        return chain.filter(exchange)
            .doFinally {
                logResponse(exchange, startTime)
            }
    }

    private fun logRequest(exchange: ServerWebExchange) {
        val request = exchange.request

        // INFO: 基本情報のみ（構造化フィールドとして出力）
        // traceIdとuserIdはMDCから自動的に取得される
        if (logger.isInfoEnabled) {
            logger.info(
                "HTTP Request",
                kv("type", "request"),
                kv("method", request.method.name()),
                kv("uri", request.uri.path)
            )
        }

        // DEBUG: ヘッダー含む
        if (logger.isDebugEnabled) {
            logger.debug(
                "HTTP Request with Headers",
                kv("type", "request"),
                kv("method", request.method.name()),
                kv("uri", request.uri.path),
                kv("headers", maskSensitiveHeaders(request.headers.toSingleValueMap()))
            )
        }

        // TRACE: ボディも含む（実装は省略、必要に応じて追加）
    }

    private fun logResponse(exchange: ServerWebExchange, startTime: Long) {
        val request = exchange.request
        val response = exchange.response
        val duration = System.currentTimeMillis() - startTime

        // INFO: 基本情報のみ（構造化フィールドとして出力）
        // traceIdとuserIdはMDCから自動的に取得される
        if (logger.isInfoEnabled) {
            logger.info(
                "HTTP Response",
                kv("type", "response"),
                kv("method", request.method.name()),
                kv("uri", request.uri.path),
                kv("status", response.statusCode?.value() ?: 0),
                kv("duration", duration)
            )
        }

        // DEBUG: ヘッダー含む
        if (logger.isDebugEnabled) {
            logger.debug(
                "HTTP Response with Headers",
                kv("type", "response"),
                kv("method", request.method.name()),
                kv("uri", request.uri.path),
                kv("status", response.statusCode?.value() ?: 0),
                kv("duration", duration),
                kv("headers", response.headers.toSingleValueMap())
            )
        }
    }

    private fun maskSensitiveHeaders(headers: Map<String, String>): Map<String, String> {
        return headers.mapValues { (key, value) ->
            if (key.lowercase() in sensitiveHeaders) "[MASKED]" else value
        }
    }
}
```

**特徴:**

- **ログレベルで出力内容を制御**:
  - `INFO`: method, uri, status, duration, traceId, userId
  - `DEBUG`: + headers（機密情報マスク済み）
  - `TRACE`: + body（実装例は省略、必要に応じて追加可能）
- Reactor Context から`traceId`と`userId`を自動取得
- JSON 形式で構造化ログ出力
- 機密ヘッダー（Authorization, Cookie, X-API-Key）の自動マスク

#### 環境別の設定

**本番環境（基本情報のみ）:**

```yaml
# application.yml
logging:
  level:
    com.example.myapp.filter.HttpLoggingFilter: INFO # method, uri, status, duration, traceId, userId
```

**開発環境（ヘッダー含む）:**

```yaml
# application-local.yml
logging:
  level:
    com.example.myapp.filter.HttpLoggingFilter: DEBUG # + headers（マスク済み）
```

**詳細デバッグ（ボディも含む）:**

```yaml
# 必要に応じて
logging:
  level:
    com.example.myapp.filter.HttpLoggingFilter: TRACE # + body（実装必要）
```

**環境変数での制御:**

```bash
# 本番環境で一時的にDEBUGログを有効化
export LOGGING_LEVEL_COM_EXAMPLE_MYAPP_FILTER_HTTPLOGGINGFILTER=DEBUG
java -jar myapp.jar
```

#### 推奨設定パターン

| 環境         | ログレベル         | 出力内容                                       | 用途                                 |
| ------------ | ------------------ | ---------------------------------------------- | ------------------------------------ |
| **本番**     | `INFO`             | method, uri, status, duration, traceId, userId | パフォーマンス監視、基本的なトレース |
| **E2E**      | `DEBUG`            | + headers（マスク済み）                        | ヘッダーの検証、API 統合テスト       |
| **ローカル** | `DEBUG` or `TRACE` | + headers, body                                | フルデバッグ、開発時の確認           |

## 外部 API ログ（WebClient）

### デフォルトの動作

WebClient は**デフォルトではリクエスト/レスポンスの詳細ログを出力しません**。

```yaml
# これだけでは何も出ない
logging:
  level:
    org.springframework.web.reactive.function.client: DEBUG
```

### 実装方法: ExchangeFilterFunction でログ出力

WebClient に`ExchangeFilterFunction`を追加して、ログを出力します。

#### WebClientLoggingFilter の実装

```kotlin
package com.example.myapp.config

import mu.KotlinLogging
import net.logstash.logback.argument.StructuredArguments.kv
import org.springframework.stereotype.Component
import org.springframework.web.reactive.function.client.ClientRequest
import org.springframework.web.reactive.function.client.ExchangeFilterFunction
import reactor.core.publisher.Mono

private val logger = KotlinLogging.logger {}

@Component
class WebClientLoggingFilter {

    private val sensitiveHeaders = setOf("authorization", "x-api-key")

    fun logRequest(): ExchangeFilterFunction {
        return ExchangeFilterFunction.ofRequestProcessor { request ->
            val startTime = System.currentTimeMillis()

            // リクエストに実行開始時刻を保存
            val newRequest = ClientRequest.from(request)
                .attribute("startTime", startTime)
                .build()

            // INFO: 基本情報のみ
            if (logger.isInfoEnabled) {
                logger.info(
                    "External API Request",
                    kv("type", "external_api_request"),
                    kv("method", request.method().name()),
                    kv("url", request.url().toString())
                )
            }

            // DEBUG: ヘッダー含む
            if (logger.isDebugEnabled) {
                logger.debug(
                    "External API Request with Headers",
                    kv("type", "external_api_request"),
                    kv("method", request.method().name()),
                    kv("url", request.url().toString()),
                    kv("headers", maskSensitiveHeaders(request.headers().toSingleValueMap()))
                )
            }

            Mono.just(newRequest)
        }
    }

    fun logResponse(): ExchangeFilterFunction {
        return ExchangeFilterFunction.ofResponseProcessor { response ->
            // リクエストから実行開始時刻を取得
            response.request().attribute("startTime").ifPresent { startTime ->
                val duration = System.currentTimeMillis() - (startTime as Long)

                // INFO: 基本情報のみ
                if (logger.isInfoEnabled) {
                    logger.info(
                        "External API Response",
                        kv("type", "external_api_response"),
                        kv("method", response.request().method().name()),
                        kv("url", response.request().url().toString()),
                        kv("status", response.statusCode().value()),
                        kv("duration", duration)
                    )
                }

                // DEBUG: ヘッダー含む
                if (logger.isDebugEnabled) {
                    logger.debug(
                        "External API Response with Headers",
                        kv("type", "external_api_response"),
                        kv("method", response.request().method().name()),
                        kv("url", response.request().url().toString()),
                        kv("status", response.statusCode().value()),
                        kv("duration", duration),
                        kv("headers", response.headers().asHttpHeaders().toSingleValueMap())
                    )
                }
            }

            Mono.just(response)
        }
    }

    private fun maskSensitiveHeaders(headers: Map<String, String>): Map<String, String> {
        return headers.mapValues { (key, value) ->
            if (key.lowercase() in sensitiveHeaders) "[MASKED]" else value
        }
    }
}
```

#### WebClientConfig に追加

```kotlin
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.reactive.function.client.WebClient

@Configuration
class WebClientConfig(
    private val loggingFilter: WebClientLoggingFilter,
) {

    @Bean
    fun externalApiWebClient(): WebClient {
        return WebClient.builder()
            .baseUrl("https://api.external.com")
            .filter(loggingFilter.logRequest())
            .filter(loggingFilter.logResponse())
            .build()
    }
}
```

### 環境別の設定

**本番環境（基本情報のみ）:**

```yaml
# application.yml
logging:
  level:
    com.example.myapp.config.WebClientLoggingFilter: INFO # method, url, status, duration
```

**開発環境（ヘッダー含む）:**

```yaml
# application-local.yml
logging:
  level:
    com.example.myapp.config.WebClientLoggingFilter: DEBUG # + headers（マスク済み）
```

**デバッグ用（完全な HTTP トラフィック）:**

開発環境でのみ、Reactor Netty の`wiretap()`を使用して完全なログを出力:

```kotlin
import io.netty.handler.logging.LogLevel
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.context.annotation.Profile
import org.springframework.http.client.reactive.ReactorClientHttpConnector
import org.springframework.web.reactive.function.client.WebClient
import reactor.netty.http.client.HttpClient
import reactor.netty.transport.logging.AdvancedByteBufFormat

@Configuration
@Profile("local")  // ローカル環境のみ
class DebugWebClientConfig {

    @Bean
    fun debugWebClient(): WebClient {
        val httpClient = HttpClient.create()
            .wiretap(
                "reactor.netty.http.client.HttpClient",
                LogLevel.DEBUG,
                AdvancedByteBufFormat.TEXTUAL  // ボディも含めて出力
            )

        return WebClient.builder()
            .clientConnector(ReactorClientHttpConnector(httpClient))
            .build()
    }
}
```

```yaml
# application-local.yml
logging:
  level:
    reactor.netty.http.client.HttpClient: DEBUG
```

**警告:** `wiretap()`はすべてのバイナリデータも含めて出力するため、**本番環境では絶対に使用しない**でください。

### 出力例

**INFO レベル（本番環境）:**

```json
// リクエスト
{
  "type": "external_api_request",
  "method": "POST",
  "url": "https://api.external.com/users"
}

// レスポンス
{
  "type": "external_api_response",
  "method": "POST",
  "url": "https://api.external.com/users",
  "status": 201,
  "duration": 234
}
```

**DEBUG レベル（開発環境）:**

```json
// リクエスト
{
  "type": "external_api_request",
  "method": "POST",
  "url": "https://api.external.com/users",
  "headers": {
    "content-type": "application/json",
    "authorization": "[MASKED]",
    "x-api-key": "[MASKED]"
  }
}

// レスポンス
{
  "type": "external_api_response",
  "method": "POST",
  "url": "https://api.external.com/users",
  "status": 201,
  "duration": 234,
  "headers": {
    "content-type": "application/json",
    "x-request-id": "abc123"
  }
}
```

### 推奨設定パターン

| 環境         | ログレベル         | 出力内容                      | 用途                                        |
| ------------ | ------------------ | ----------------------------- | ------------------------------------------- |
| **本番**     | `INFO`             | method, url, status, duration | 外部 API 呼び出しの監視、パフォーマンス測定 |
| **E2E**      | `DEBUG`            | + headers（マスク済み）       | API 統合テストの検証                        |
| **ローカル** | `DEBUG` or wiretap | + headers, body               | フルデバッグ、外部 API 開発時の確認         |

**メリット:**

- 外部 API へのリクエスト/レスポンスを記録
- 実行時間を測定してパフォーマンス監視
- ログレベルで簡単に制御
- 機密ヘッダー（Authorization, API Key 等）の自動マスク
- JSON 形式で構造化されたログ出力

## ログ収集とモニタリング

### CloudWatch Logs

```yaml
# docker-compose.yml（ローカル開発用）
version: "3.8"
services:
  app:
    image: myapp:latest
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### Datadog

```yaml
# application.yml
management:
  metrics:
    export:
      datadog:
        api-key: ${DATADOG_API_KEY}
        application-key: ${DATADOG_APP_KEY}
```

### Elasticsearch + Kibana

JSON 形式のログを Fluentd/Filebeat 経由で Elasticsearch に送信し、Kibana で可視化します。

```yaml
# filebeat.yml
filebeat.inputs:
  - type: container
    paths:
      - "/var/lib/docker/containers/*/*.log"

processors:
  - decode_json_fields:
      fields: ["message"]
      target: ""
      overwrite_keys: true

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "myapp-logs-%{+yyyy.MM.dd}"
```

## ベストプラクティス

### 1. 構造化されたメッセージ

```kotlin
// ❌ 非推奨: 文字列結合
logger.info("User created: " + user.id + ", email: " + user.email)

// ✅ 推奨: 構造化されたメッセージ
logger.info { "User created: id=${user.id}, email=${user.email}" }
```

### 2. 機密情報のマスキング

```kotlin
// ❌ 非推奨: パスワードをログ出力
logger.debug { "Login attempt: email=$email, password=$password" }

// ✅ 推奨: 機密情報をマスク
logger.debug { "Login attempt: email=$email" }

// data classのtoString()から機密情報を除外
data class UserCredentials(
    val username: String,
    private val password: String,
) {
    override fun toString(): String = "UserCredentials(username=$username, password=***)"
}
```

### 3. 例外ログの適切な出力

```kotlin
// ❌ 非推奨: スタックトレースを文字列化
logger.error { "Error: ${e.stackTraceToString()}" }

// ✅ 推奨: 例外オブジェクトを渡す
logger.error(e) { "Failed to process order: orderId=$orderId" }
```

### 4. ログレベルの適切な使用

```kotlin
@Service
class PaymentService {

    suspend fun processPayment(amount: Money) {
        // INFO: ビジネスイベント
        logger.info { "Processing payment: amount=$amount" }

        try {
            // DEBUG: 開発時のデバッグ情報
            logger.debug { "Payment gateway request: ${buildRequest(amount)}" }

            val result = paymentGateway.charge(amount)

            // INFO: 成功イベント
            logger.info { "Payment successful: transactionId=${result.id}" }

        } catch (e: TemporaryPaymentException) {
            // WARN: リトライ可能なエラー
            logger.warn(e) { "Payment temporarily failed, will retry: amount=$amount" }
            throw e
        } catch (e: PaymentException) {
            // ERROR: 予期しないエラー
            logger.error(e) { "Payment failed permanently: amount=$amount" }
            throw e
        }
    }
}
```

### 5. パフォーマンスへの配慮

```kotlin
// ❌ 非推奨: ログレベルに関係なく処理を実行
logger.debug(expensiveOperation())

// ✅ 推奨: ラムダ式で遅延評価
logger.debug { "Result: ${expensiveOperation()}" }
```

## まとめ

- **JSON 形式で統一** - すべての環境で構造化ログ
- **TraceId で追跡** - New Relic とログを紐付ける
- **適切なログレベル** - 環境ごとに最適化
- **MDC でコンテキスト情報** - traceId、userId を自動付与
- **補完関係** - New Relic（処理フロー）とログ（ビジネス詳細）を組み合わせる
