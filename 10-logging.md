# ロギング

## 基本方針

1. **JSON形式で統一** - すべての環境でLogstashEncoderを使用
2. **構造化ログ** - 検索・分析しやすい形式で出力
3. **TraceIdで追跡** - リクエスト全体を通してログを紐付ける
4. **適切なログレベル** - 環境ごとに最適なレベルを設定
5. **MDCでコンテキスト情報を付与** - TraceId、UserIdなどを自動付与

## Logback設定

### 依存関係の追加

```kotlin
// build.gradle.kts
dependencies {
    implementation("net.logstash.logback:logstash-logback-encoder:7.4")
    implementation("io.github.microutils:kotlin-logging-jvm:3.0.5")
}
```

### Logback設定ファイル

**全環境でJSON形式のログを出力します。**

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

### Kotlin Loggingの使用（推奨）

KotlinではSLF4Jを直接使わず、**kotlin-logging（KotlinLogging）** を使用します。

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

### KotlinLoggingを推奨する理由

#### 1. 遅延評価による性能向上

```kotlin
// ❌ SLF4J: ログレベルが無効でも文字列連結が実行される
logger.debug("User data: ${expensiveOperation()}")

// ✅ KotlinLogging: DEBUGレベルが無効なら expensiveOperation() は呼ばれない
logger.debug { "User data: ${expensiveOperation()}" }
```

#### 2. Kotlinイディオムに沿った記述

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

| レベル | 用途 | 例 |
|--------|------|-----|
| **ERROR** | 予期しないエラー、システム障害 | 外部API障害、DB接続エラー、予期しない例外 |
| **WARN** | 注意が必要な状況、潜在的な問題 | リトライ発生、非推奨API使用、ビジネスルール違反 |
| **INFO** | 重要なビジネスイベント | ユーザー作成、注文完了、重要な状態変更 |
| **DEBUG** | 詳細なデバッグ情報（開発用） | クエリパラメータ、中間結果、内部状態 |

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

## New Relic TraceIdのMDC連携

### New Relicの仕組みとログ連携の必要性

New Relicは**Java Agent**として動作し、バイトコードを実行時に書き換えて自動的にトレース情報を収集します。

```bash
# アプリケーション起動時
java -javaagent:/path/to/newrelic.jar -jar myapp.jar
```

**New Relicが自動収集する情報:**
- 処理フロー（どのメソッドが呼ばれたか）
- 各処理の実行時間
- DB クエリ、HTTP リクエストの詳細
- TraceId と SpanId（階層構造）

**New Relicが収集しない情報:**
- **アプリケーションログの内容**（`logger.info { }` などの出力）
- ビジネスロジックの詳細
- デバッグ用の変数値
- エラーの詳細なコンテキスト

そのため、**New Relicとアプリケーションログは補完関係**にあります：

| 用途 | New Relic | アプリケーションログ |
|------|-----------|---------------------|
| **どこが遅いか** | ✅ 処理時間を可視化 | ❌ |
| **何が呼ばれたか** | ✅ メソッド呼び出しを記録 | ❌ |
| **なぜ失敗したか** | ❌ | ✅ エラーメッセージと詳細 |
| **ビジネスデータ** | ❌ | ✅ 注文ID、金額などの詳細 |

**TraceIdをログに含めることで、New Relicのトレース画面とログを相互に行き来して調査できます。**

### 依存関係の追加

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.newrelic.agent.java:newrelic-api:8.8.0")
}
```

### WebFilterでのMDC設定

**重要:** TraceIdのみをMDCに設定します。SpanIdは処理ごとに変わるため、Filterレベルで設定すると意味がありません。

```kotlin
import com.newrelic.api.agent.NewRelic
import kotlinx.coroutines.slf4j.MDCContext
import kotlinx.coroutines.withContext
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
class NewRelicMdcFilter : WebFilter {

    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        return Mono.deferContextual { contextView ->
            // New RelicからTraceIdのみを取得
            // TraceIdはリクエスト全体で1つなので、Filterで設定する意味がある
            val traceId = NewRelic.getAgent().traceMetadata.traceId

            // MDCにTraceIdのみを設定
            // SpanIdは処理ごとに変わるため、Filterレベルでは設定しない
            MDC.put("traceId", traceId)

            try {
                chain.filter(exchange)
            } finally {
                // リクエスト処理後にMDCをクリア
                MDC.remove("traceId")
            }
        }
    }
}
```

**TraceIdとSpanIdの違い:**

| 項目 | TraceId | SpanId |
|------|---------|--------|
| **粒度** | リクエスト全体 | 個々の処理単位 |
| **数** | リクエストごとに1つ | リクエスト内で複数 |
| **用途** | 関連する処理をまとめる | 処理の内訳を追跡 |
| **ログへの含め方** | Filterで設定 | 各処理で動的に取得（通常は不要） |

SpanIdの階層構造はNew Relicが自動的に管理するため、アプリケーションログに含める必要はありません。

### WebFlux環境でのMDC伝搬の課題

**重要:** WebFluxはイベント駆動・非同期処理のため、従来のMDC設定では以下の問題があります：

1. **リクエスト境界の喪失**: Reactorのスレッドプールは複数リクエストで共有されるため、MDCが混在する
2. **Contextの切り替え**: `subscribeOn()`や`publishOn()`でスレッドが変わるとMDCが失われる
3. **タイミング問題**: Scheduler Hookはタスクスケジュール時に呼ばれるが、リクエストコンテキストとは独立している

これらの理由から、**Reactor Context**を使ってリクエストスコープでtraceIdを伝搬させる必要があります。

### 実装方法: Reactor Contextとカスタムログプロバイダー

#### 1. WebFilterでReactor Contextに設定

```kotlin
import com.newrelic.api.agent.NewRelic
import org.springframework.core.Ordered
import org.springframework.core.annotation.Order
import org.springframework.stereotype.Component
import org.springframework.web.server.ServerWebExchange
import org.springframework.web.server.WebFilter
import org.springframework.web.server.WebFilterChain
import reactor.core.publisher.Mono

@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
class TraceIdContextFilter : WebFilter {

    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        val traceId = NewRelic.getAgent().traceMetadata.traceId

        // Reactor ContextにtraceIdを設定（リクエストスコープで確実に伝搬される）
        return chain.filter(exchange)
            .contextWrite { context ->
                context.put("traceId", traceId)
            }
    }
}
```

#### 2. カスタムLogstashプロバイダーでReactor ContextからtraceIdを取得

ログ出力時に毎回`withContext`でMDCを設定するのは煩雑なため、Logbackの拡張機能を使ってReactor Contextから直接traceIdを取得します。

**依存関係の追加:**

```kotlin
// build.gradle.kts
dependencies {
    implementation("net.logstash.logback:logstash-logback-encoder:7.4")
    implementation("io.projectreactor:reactor-core")
}
```

**Logback設定:**

```xml
<!-- src/main/resources/logback-spring.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

    <!-- JSON形式のAppender -->
    <appender name="JSON_CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- Reactor ContextからtraceIdを取得するカスタムプロバイダー -->
            <provider class="com.example.myapp.logging.ReactorContextJsonProvider"/>

            <!-- タイムスタンプ形式 -->
            <timestampPattern>yyyy-MM-dd'T'HH:mm:ss.SSSZZ</timestampPattern>

            <!-- カスタムフィールド -->
            <customFields>{"application":"my-app"}</customFields>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="JSON_CONSOLE"/>
    </root>
</configuration>
```

**カスタムプロバイダーの実装:**

```kotlin
package com.example.myapp.logging

import ch.qos.logback.classic.spi.ILoggingEvent
import com.fasterxml.jackson.core.JsonGenerator
import net.logstash.logback.composite.AbstractFieldJsonProvider
import net.logstash.logback.composite.JsonWritingUtils
import reactor.core.publisher.Mono

class ReactorContextJsonProvider : AbstractFieldJsonProvider<ILoggingEvent>() {

    override fun writeTo(generator: JsonGenerator, event: ILoggingEvent) {
        // Reactor ContextからtraceIdを取得してJSONに追加
        try {
            Mono.deferContextual { contextView ->
                val traceId = contextView.getOrDefault<String>("traceId", "")
                if (traceId.isNotEmpty()) {
                    JsonWritingUtils.writeStringField(generator, "traceId", traceId)
                }
                Mono.empty<Void>()
            }.block() // ログ出力時は同期的に取得
        } catch (e: Exception) {
            // Reactor Context外で実行された場合は何もしない
        }
    }
}
```

**使用例（アプリケーションコード）:**

```kotlin
import mu.KotlinLogging
import org.springframework.stereotype.Service

private val logger = KotlinLogging.logger {}

@Service
class UserService(
    private val userRepository: UserRepository,
) {
    suspend fun findById(id: UserId): User? {
        // 特別なwithContextは不要。通常通りログ出力するだけ
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

### 非推奨: Reactor Scheduler Hook

以下のようなScheduler Hookは**WebFlux環境では使用しないでください**：

```kotlin
// ❌ 非推奨: リクエスト境界を越えてMDCが混在する可能性がある
@Bean
fun reactorSchedulerHook() {
    Schedulers.onScheduleHook("mdc") { runnable ->
        val contextMap = MDC.getCopyOfContextMap()
        Runnable {
            contextMap?.let { MDC.setContextMap(it) }
            try {
                runnable.run()
            } finally {
                MDC.clear()
            }
        }
    }
}
```

**理由:**
- スレッドプールは複数のリクエストで共有されるため、MDCが別リクエストと混在する
- Reactor Contextの方がリクエストスコープで確実に伝搬される
- ログ出力のたびにMDC設定を書く必要がなく、コードがシンプルになる

### ログ出力例

```json
{
  "@timestamp": "2024-01-15T10:30:45.123+09:00",
  "level": "INFO",
  "logger": "com.example.myapp.application.UserService",
  "message": "Finding user: 123",
  "traceId": "abc123def456",
  "userId": "123",
  "application": "my-app",
  "env": "production"
}
```

**ログに含まれる情報:**
- `@timestamp`: ログ出力日時
- `level`: ログレベル
- `logger`: ロガー名（クラス名）
- `message`: ログメッセージ
- `traceId`: New Relicとログを紐付けるためのID
- `userId`: アプリケーション固有のコンテキスト情報
- `application`: アプリケーション名
- `env`: 環境名

## New Relicとログの連携

### 調査フロー例: 「注文処理が失敗した」問題のデバッグ

**1. New Relicでトレースを確認** - どこで時間がかかっているかを特定

```
TraceId: abc123
├─ Controller.createOrder - 2500ms
│  ├─ OrderService.processOrder - 2400ms
│  │  ├─ Database.query - 50ms
│  │  ├─ PaymentGateway.charge - 2300ms ← ここが遅い！
│  │  └─ Database.update - 20ms
```

**わかること:** PaymentGatewayで2.3秒かかっている

**わからないこと:** なぜ失敗したのか、どんなエラーか

**2. TraceIdでログを検索** - ビジネスロジックの詳細を確認

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

**わかること:** 注文ID、金額、失敗理由（カード残高不足）

**3. 両方を組み合わせて完全な情報を得る**
- New Relic: 処理フローと性能ボトルネック
- ログ: ビジネスロジックの詳細とエラー原因

### New Relicとの統合機能

New Relicの一部のプランでは、ログを直接New Relicに送信して統合表示できます。その場合でも、TraceIdがログに含まれていることで自動的に紐付けが行われます。

### New Relicが無効な場合のフォールバック

**注意:** New Relicエージェントが有効でない場合（ローカル開発環境など）、`traceId`は空文字列になる可能性があります。その場合は独自のトレースIDを生成することを検討してください。

```kotlin
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
class TraceIdMdcFilter : WebFilter {

    override fun filter(exchange: ServerWebExchange, chain: WebFilterChain): Mono<Void> {
        return Mono.deferContextual { contextView ->
            // New Relicが無効な場合のフォールバック
            val traceId = try {
                NewRelic.getAgent().traceMetadata.traceId.takeIf { it.isNotBlank() }
            } catch (e: Exception) {
                null
            } ?: UUID.randomUUID().toString()

            MDC.put("traceId", traceId)

            try {
                chain.filter(exchange)
            } finally {
                MDC.remove("traceId")
            }
        }
    }
}
```

## カスタムMDCフィールドの追加

### UserIdの追加

認証済みユーザーのIDもReactor Contextに追加し、ログに含めます。

#### 1. WebFilterでReactor ContextにuserIdを追加

```kotlin
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
@Order(Ordered.HIGHEST_PRECEDENCE + 1)  // TraceIdContextFilterの次に実行
class UserIdContextFilter : WebFilter {

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

                // Reactor ContextにuserIdを設定
                chain.filter(exchange)
                    .contextWrite { context ->
                        userId?.let { context.put("userId", it) } ?: context
                    }
            }
            .switchIfEmpty(chain.filter(exchange))
    }
}
```

#### 2. ReactorContextJsonProviderでuserIdも取得

```kotlin
package com.example.myapp.logging

import ch.qos.logback.classic.spi.ILoggingEvent
import com.fasterxml.jackson.core.JsonGenerator
import net.logstash.logback.composite.AbstractFieldJsonProvider
import net.logstash.logback.composite.JsonWritingUtils
import reactor.core.publisher.Mono

class ReactorContextJsonProvider : AbstractFieldJsonProvider<ILoggingEvent>() {

    override fun writeTo(generator: JsonGenerator, event: ILoggingEvent) {
        // Reactor ContextからtraceIdとuserIdを取得してJSONに追加
        try {
            Mono.deferContextual { contextView ->
                // traceIdを取得
                val traceId = contextView.getOrDefault<String>("traceId", "")
                if (traceId.isNotEmpty()) {
                    JsonWritingUtils.writeStringField(generator, "traceId", traceId)
                }

                // userIdを取得
                val userId = contextView.getOrDefault<String>("userId", "")
                if (userId.isNotEmpty()) {
                    JsonWritingUtils.writeStringField(generator, "userId", userId)
                }

                Mono.empty<Void>()
            }.block() // ログ出力時は同期的に取得
        } catch (e: Exception) {
            // Reactor Context外で実行された場合は何もしない
        }
    }
}
```

#### 3. 使用例（アプリケーションコード）

```kotlin
import mu.KotlinLogging
import org.springframework.stereotype.Service

private val logger = KotlinLogging.logger {}

@Service
class OrderService(
    private val orderRepository: OrderRepository,
) {
    suspend fun createOrder(request: CreateOrderRequest): Order {
        // userIdは自動的にログに含まれる（Reactor Contextから取得される）
        logger.info { "Creating order: amount=${request.amount}" }

        val order = Order.create(request)
        return orderRepository.save(order.toEntity()).toDomain()
    }
}
```

**ログ出力例:**

```json
{
  "@timestamp": "2024-01-15T10:30:45.123+09:00",
  "level": "INFO",
  "logger": "com.example.myapp.application.OrderService",
  "message": "Creating order: amount=5000",
  "traceId": "abc123def456",
  "userId": "user-12345",
  "application": "my-app"
}
```

## HTTPリクエスト/レスポンスのログ出力

### デフォルトの動作

Spring WebFluxでは**デフォルトではリクエスト/レスポンスの詳細ログは出力されません**。

```yaml
# Spring WebFluxのログレベルをDEBUGにした場合
logging:
  level:
    org.springframework.web.server.adapter.HttpWebHandlerAdapter: DEBUG
```

**出力される内容（最小限・JSON形式）:**

```json
{
  "level": "DEBUG",
  "logger": "org.springframework.web.server.adapter.HttpWebHandlerAdapter",
  "message": "HTTP GET \"/api/users/123\"",
  "traceId": "abc123"
}
```

- HTTPメソッド
- パス
- ステータスコード

**出力されない内容:**
- リクエストヘッダー（`Authorization`等）
- リクエスト/レスポンスボディ
- クエリパラメータの値
- **実行時間**（duration）
- **構造化されたフィールド**（method, uri, statusが別フィールドとして分離されていない）

### 実装方法: 独自WebFilterでログ出力

詳細なHTTPログをJSON形式で出力するには、カスタムWebFilterを実装します。

#### HttpLoggingFilterの実装

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

        // Reactor ContextからtraceIdとuserIdを取得
        Mono.deferContextual { ctx ->
            val traceId = ctx.getOrDefault<String>("traceId", "")
            val userId = ctx.getOrDefault<String>("userId", "")

            // INFO: 基本情報のみ（構造化フィールドとして出力）
            if (logger.isInfoEnabled) {
                logger.info(
                    "HTTP Request",
                    kv("type", "request"),
                    kv("method", request.method.name()),
                    kv("uri", request.uri.path),
                    *if (traceId.isNotEmpty()) arrayOf(kv("traceId", traceId)) else emptyArray(),
                    *if (userId.isNotEmpty()) arrayOf(kv("userId", userId)) else emptyArray()
                )
            }

            // DEBUG: ヘッダー含む
            if (logger.isDebugEnabled) {
                logger.debug(
                    "HTTP Request",
                    kv("type", "request"),
                    kv("method", request.method.name()),
                    kv("uri", request.uri.path),
                    kv("headers", maskSensitiveHeaders(request.headers.toSingleValueMap())),
                    *if (traceId.isNotEmpty()) arrayOf(kv("traceId", traceId)) else emptyArray(),
                    *if (userId.isNotEmpty()) arrayOf(kv("userId", userId)) else emptyArray()
                )
            }

            // TRACE: ボディも含む（実装は省略、必要に応じて追加）

            Mono.empty<Void>()
        }.subscribe()
    }

    private fun logResponse(exchange: ServerWebExchange, startTime: Long) {
        val request = exchange.request
        val response = exchange.response
        val duration = System.currentTimeMillis() - startTime

        // Reactor ContextからtraceIdとuserIdを取得
        Mono.deferContextual { ctx ->
            val traceId = ctx.getOrDefault<String>("traceId", "")
            val userId = ctx.getOrDefault<String>("userId", "")

            // INFO: 基本情報のみ（構造化フィールドとして出力）
            if (logger.isInfoEnabled) {
                logger.info(
                    "HTTP Response",
                    kv("type", "response"),
                    kv("method", request.method.name()),
                    kv("uri", request.uri.path),
                    kv("status", response.statusCode?.value() ?: 0),
                    kv("duration", duration),
                    *if (traceId.isNotEmpty()) arrayOf(kv("traceId", traceId)) else emptyArray(),
                    *if (userId.isNotEmpty()) arrayOf(kv("userId", userId)) else emptyArray()
                )
            }

            // DEBUG: ヘッダー含む
            if (logger.isDebugEnabled) {
                logger.debug(
                    "HTTP Response",
                    kv("type", "response"),
                    kv("method", request.method.name()),
                    kv("uri", request.uri.path),
                    kv("status", response.statusCode?.value() ?: 0),
                    kv("duration", duration),
                    kv("headers", response.headers.toSingleValueMap()),
                    *if (traceId.isNotEmpty()) arrayOf(kv("traceId", traceId)) else emptyArray(),
                    *if (userId.isNotEmpty()) arrayOf(kv("userId", userId)) else emptyArray()
                )
            }

            Mono.empty<Void>()
        }.subscribe()
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
- Reactor Contextから`traceId`と`userId`を自動取得
- JSON形式で構造化ログ出力
- 機密ヘッダー（Authorization, Cookie, X-API-Key）の自動マスク

#### 環境別の設定

**本番環境（基本情報のみ）:**

```yaml
# application.yml
logging:
  level:
    com.example.myapp.filter.HttpLoggingFilter: INFO  # method, uri, status, duration, traceId, userId
```

**開発環境（ヘッダー含む）:**

```yaml
# application-local.yml
logging:
  level:
    com.example.myapp.filter.HttpLoggingFilter: DEBUG  # + headers（マスク済み）
```

**詳細デバッグ（ボディも含む）:**

```yaml
# 必要に応じて
logging:
  level:
    com.example.myapp.filter.HttpLoggingFilter: TRACE  # + body（実装必要）
```

**環境変数での制御:**

```bash
# 本番環境で一時的にDEBUGログを有効化
export LOGGING_LEVEL_COM_EXAMPLE_MYAPP_FILTER_HTTPLOGGINGFILTER=DEBUG
java -jar myapp.jar
```

#### 出力例

**INFO レベル（本番環境）:**

```json
// リクエスト
{
  "type": "request",
  "method": "GET",
  "uri": "/api/users/123",
  "traceId": "new-relic-trace-123",
  "userId": "user-456"
}

// レスポンス
{
  "type": "response",
  "method": "GET",
  "uri": "/api/users/123",
  "status": 200,
  "duration": 45,
  "traceId": "new-relic-trace-123",
  "userId": "user-456"
}
```

**DEBUG レベル（開発環境）:**

```json
// リクエスト
{
  "type": "request",
  "method": "POST",
  "uri": "/api/users",
  "headers": {
    "content-type": "application/json",
    "authorization": "[MASKED]",
    "user-agent": "Mozilla/5.0..."
  },
  "traceId": "new-relic-trace-123",
  "userId": "user-456"
}

// レスポンス
{
  "type": "response",
  "method": "POST",
  "uri": "/api/users",
  "status": 201,
  "duration": 123,
  "headers": {
    "content-type": "application/json",
    "location": "/api/users/123"
  },
  "traceId": "new-relic-trace-123",
  "userId": "user-456"
}
```

#### 推奨設定パターン

| 環境 | ログレベル | 出力内容 | 用途 |
|------|-----------|---------|------|
| **本番** | `INFO` | method, uri, status, duration, traceId, userId | パフォーマンス監視、基本的なトレース |
| **E2E** | `DEBUG` | + headers（マスク済み） | ヘッダーの検証、API統合テスト |
| **ローカル** | `DEBUG` or `TRACE` | + headers, body | フルデバッグ、開発時の確認 |

**メリット:**
- 依存関係不要（標準的なWebFilter）
- traceId/userIdがReactor Contextから確実に取得される
- ログレベルで簡単に出力内容を制御
- 環境変数で動的に切り替え可能
- 実装がシンプルで理解しやすい
- JSON形式で構造化されたログ出力

## 外部APIログ（WebClient）

### デフォルトの動作

WebClientは**デフォルトではリクエスト/レスポンスの詳細ログを出力しません**。

```yaml
# これだけでは何も出ない
logging:
  level:
    org.springframework.web.reactive.function.client: DEBUG
```

### 実装方法: ExchangeFilterFunctionでログ出力

WebClientに`ExchangeFilterFunction`を追加して、ログを出力します。

#### WebClientLoggingFilterの実装

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

#### WebClientConfigに追加

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
    com.example.myapp.config.WebClientLoggingFilter: INFO  # method, url, status, duration
```

**開発環境（ヘッダー含む）:**

```yaml
# application-local.yml
logging:
  level:
    com.example.myapp.config.WebClientLoggingFilter: DEBUG  # + headers（マスク済み）
```

**デバッグ用（完全なHTTPトラフィック）:**

開発環境でのみ、Reactor Nettyの`wiretap()`を使用して完全なログを出力:

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

| 環境 | ログレベル | 出力内容 | 用途 |
|------|-----------|---------|------|
| **本番** | `INFO` | method, url, status, duration | 外部API呼び出しの監視、パフォーマンス測定 |
| **E2E** | `DEBUG` | + headers（マスク済み） | API統合テストの検証 |
| **ローカル** | `DEBUG` or wiretap | + headers, body | フルデバッグ、外部API開発時の確認 |

**メリット:**
- 外部APIへのリクエスト/レスポンスを記録
- 実行時間を測定してパフォーマンス監視
- ログレベルで簡単に制御
- 機密ヘッダー（Authorization, API Key等）の自動マスク
- JSON形式で構造化されたログ出力

## ログ収集とモニタリング

### CloudWatch Logs

```yaml
# docker-compose.yml（ローカル開発用）
version: '3.8'
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

JSON形式のログをFluentd/Filebeat経由でElasticsearchに送信し、Kibanaで可視化します。

```yaml
# filebeat.yml
filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'

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

- **JSON形式で統一** - すべての環境で構造化ログ
- **TraceIdで追跡** - New Relicとログを紐付ける
- **適切なログレベル** - 環境ごとに最適化
- **MDCでコンテキスト情報** - traceId、userIdを自動付与
- **補完関係** - New Relic（処理フロー）とログ（ビジネス詳細）を組み合わせる
