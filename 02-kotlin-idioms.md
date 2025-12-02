# Kotlin イディオム

## 概要

Kotlin は Java と比較して表現の自由度が高い。そのため、チーム内で書き方を統一しないとコードの一貫性が失われやすい。本ドキュメントでは、本プロジェクトで採用する Kotlin の書き方を定める。

## トップレベル関数 vs object

### 方針: トップレベル関数を優先する

状態を持たないユーティリティ関数は、`object`ではなくトップレベル関数として定義する。

```kotlin
// 推奨: トップレベル関数
// StringExtensions.kt
fun String.toSlug(): String =
    this.lowercase().replace(" ", "-")

fun generateRandomId(): String =
    UUID.randomUUID().toString()
```

```kotlin
// 非推奨: objectでのラップ
object StringUtils {
    fun toSlug(value: String): String =
        value.lowercase().replace(" ", "-")

    fun generateRandomId(): String =
        UUID.randomUUID().toString()
}
```

### 理由

- Java の`static`メソッドに相当する機能として、Kotlin ではトップレベル関数が自然
- 呼び出し時に不要なプレフィックス（`StringUtils.`）が不要
- 拡張関数との組み合わせでより直感的な API になる

### object を使用するケース

以下の場合は`object`を使用する。

```kotlin
// シングルトンとして状態を持つ場合
object ApplicationMetrics {
    private val requestCount = AtomicLong(0)

    fun incrementRequestCount() {
        requestCount.incrementAndGet()
    }

    fun getRequestCount(): Long = requestCount.get()
}

// インターフェースの実装を提供する場合
object EmptyUserValidator : UserValidator {
    override fun validate(user: User): ValidationResult = ValidationResult.Success
}

// companion objectでファクトリメソッドを提供する場合
data class UserId(val value: Long) {
    companion object {
        fun fromString(value: String): UserId = UserId(value.toLong())
    }
}
```

## data class / sealed class / value class

### data class

不変のデータを保持するクラスに使用する。`equals()`, `hashCode()`, `toString()`, `copy()`が自動生成される。

```kotlin
// DTOに使用
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
)

// ドメインモデルに使用
data class User(
    val id: UserId,
    val name: UserName,
    val email: Email,
    val createdAt: Instant,
)
```

**注意点:**

- プロパティはすべて`val`（不変）にする
- 継承が必要な場合は通常の class を使用する

### sealed class / sealed interface

限定された型のバリエーションを表現する場合に使用する。`when`式で網羅性チェックが可能になる。

```kotlin
// 処理結果の表現
sealed interface Result<out T> {
    data class Success<T>(val value: T) : Result<T>
    data class Failure(val error: AppError) : Result<Nothing>
}

// ドメインイベント
sealed interface UserEvent {
    data class Created(val userId: UserId, val timestamp: Instant) : UserEvent
    data class Updated(val userId: UserId, val changes: UserChanges) : UserEvent
    data class Deleted(val userId: UserId) : UserEvent
}

// 使用例: whenで網羅性が保証される
fun handleEvent(event: UserEvent): Unit = when (event) {
    is UserEvent.Created -> notifyUserCreated(event.userId)
    is UserEvent.Updated -> syncUserChanges(event.changes)
    is UserEvent.Deleted -> cleanupUserData(event.userId)
    // 新しいバリアントを追加するとコンパイルエラーになる
}
```

**sealed class と sealed interface の使い分け:**

- 共通の状態やロジックを持たせたい場合: `sealed class`
- 単なる型の分類のみの場合: `sealed interface`（推奨）

### value class（inline class）

プリミティブ型をラップして型安全性を高める場合に使用する。ランタイムオーバーヘッドがないが、**過剰な利用は控える**。

```kotlin
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "Invalid email format" }
    }
}

@JvmInline
value class Money(val amount: BigDecimal)

// 型による区別で引数の取り違えを防ぐ
fun transferMoney(from: UserId, to: UserId, amount: Money) { ... }

// コンパイルエラー: 引数の順序間違いを検出
transferMoney(amount = money, from = userId1, to = userId2)  // 名前付き引数で明確化
```

**使用基準（以下をすべて満たす場合のみ採用を検討）:**

- 型の混同による重大なバグのリスクがある
- 複雑なドメインルールがあり、型で表現する価値が高い
- **R2DBC や Jackson のカスタムコンバーターを実装するコストを許容できる**

**注意点:**

- R2DBC で使用する場合、プリミティブ型への変換コンバーターが必要
- Jackson でのシリアライズ/デシリアライズ時にカスタム設定が必要
- 多くの場合、**プリミティブ型で十分**（Request DTO でバリデーションを完全に行う前提）

**推奨アプローチ:**

基本的には**プリミティブ型を使用**し、上記の使用基準をすべて満たす場合のみ value class を採用する。

## Nullable の扱い

### 方針: null を避け、明示的な型で表現する

```kotlin
// 非推奨: nullableで存在しないことを表現
fun findById(id: UserId): User?

// 推奨: 意図が明確な戻り値型
suspend fun findById(id: UserId): User?  // 単純な検索はnullable許容

// より複雑なケースではResultやsealedで表現
sealed interface FindUserResult {
    data class Found(val user: User) : FindUserResult
    data object NotFound : FindUserResult
    data class Error(val cause: Throwable) : FindUserResult
}
```

### null 安全演算子の使い分け

```kotlin
// ?.let: nullでない場合のみ処理を実行
user?.let { sendWelcomeEmail(it) }

// ?: (elvis): デフォルト値の提供
val name = user?.name ?: "Anonymous"

// ?: throw: nullの場合は例外
val user = userRepository.findById(id)
    ?: throw UserNotFoundException(id)

// ?: return: 早期リターン
fun process(userId: UserId?): ProcessResult {
    val id = userId ?: return ProcessResult.InvalidInput
    // 以降、idはnon-null
    ...
}
```

### !! 演算子の禁止

`!!`（非 null アサーション）は原則として使用禁止。使用する場合はコメントで理由を明記する。

```kotlin
// 禁止: 理由のない!!
val name = user!!.name

// 許容: 明確な理由がある場合（ただし避けるべき）
// フレームワークの制約でnullableになるが、この時点では必ずnon-null
val context = SecurityContextHolder.getContext().authentication!!  // Spring Securityの仕様
```

## スコープ関数

スコープ関数は通常の拡張関数として実装されており、レシーバー付きラムダを活用している。適切に使うとコードが簡潔になるが、乱用すると可読性が下がる。

### 使い分けの基準

| 関数    | レシーバー | 戻り値       | 主な用途                         |
| ------- | ---------- | ------------ | -------------------------------- |
| `let`   | `it`       | ラムダの結果 | nullable 処理、変換              |
| `run`   | `this`     | ラムダの結果 | オブジェクトの設定と結果取得     |
| `apply` | `this`     | レシーバー   | オブジェクトの初期化             |
| `also`  | `it`       | レシーバー   | 副作用（ログ出力など）           |
| `with`  | `this`     | ラムダの結果 | 非 null オブジェクトへの複数操作 |

### 推奨パターン

```kotlin
// let: nullableの処理
user?.let { sendNotification(it) }

// let: 変換処理
val userIds = users.map { it.id.value.let(::UserId) }

// apply: ビルダーパターン的な初期化
val request = Request.Builder().apply {
    url(endpoint)
    header("Authorization", "Bearer $token")
    post(body)
}.build()

// also: 副作用（ログ、デバッグ）
return userRepository.save(user).also {
    logger.info { "User saved: ${it.id}" }
}

// run: 初期化と結果取得
val result = service.run {
    initialize()
    process(input)
}

// with: 非nullオブジェクトへの複数操作
val csv = with(user) {
    "$id,$name,$email"
}
```

### 避けるべきパターン

```kotlin
// 非推奨: ネストしたスコープ関数
user?.let { u ->
    u.address?.let { addr ->
        addr.city?.let { city ->
            processCity(city)
        }
    }
}

// 推奨: 早期リターンまたはフラットな条件
val city = user?.address?.city ?: return
processCity(city)

// 非推奨: 意味のないスコープ関数
val result = value.let { it.toString() }

// 推奨: 直接呼び出し
val result = value.toString()
```

## 拡張関数

### 使用する場面

```kotlin
// 既存クラスへの機能追加
fun String.toSlug(): String =
    this.lowercase()
        .replace(Regex("[^a-z0-9\\s-]"), "")
        .replace(Regex("\\s+"), "-")

// ドメイン固有の変換
fun UserEntity.toDomain(): User = User(
    id = UserId(this.id),
    name = UserName(this.name),
    email = Email(this.email),
)

// 可読性向上のためのDSL的な表現
infix fun Money.plus(other: Money): Money =
    Money(this.amount + other.amount)

val total = price plus tax
```

### 避けるべき使い方

```kotlin
// 非推奨: 副作用を持つ拡張関数
fun User.saveToDatabase() {
    userRepository.save(this)  // 副作用が隠れている
}

// 非推奨: 過度に汎用的な拡張
fun <T> T.log(): T {
    println(this)
    return this
}

// 非推奨: レシーバーと関係の薄い処理
fun String.sendEmail() { ... }  // Stringがメールアドレスとは限らない
```

### 拡張関数の配置

```kotlin
// ファイル構成
common/
└── extension/
    ├── StringExtensions.kt    # String型への拡張
    ├── InstantExtensions.kt   # 時刻関連の拡張
    └── FlowExtensions.kt      # Coroutines Flow関連の拡張

user/
└── UserExtensions.kt          # User機能固有の拡張（ドメイン変換など）
```

## コレクション操作

### 推奨パターン

```kotlin
// シーケンス: 大量データの遅延評価
users.asSequence()
    .filter { it.isActive }
    .map { it.toResponse() }
    .take(100)
    .toList()

// 適切な終端操作の選択
val hasAdmin = users.any { it.role == Role.ADMIN }      // 見つかったら即終了
val allActive = users.all { it.isActive }               // falseが見つかったら即終了
val firstAdmin = users.firstOrNull { it.role == Role.ADMIN }

// groupByとassociateの使い分け
val usersByRole = users.groupBy { it.role }             // Map<Role, List<User>>
val userById = users.associateBy { it.id }              // Map<UserId, User>

// destructuring
users.forEach { (id, name, email) ->
    println("$id: $name <$email>")
}
```

### 避けるべきパターン

```kotlin
// 非推奨: 不要な中間リスト生成
users.filter { ... }.map { ... }.filter { ... }.first()

// 推奨: シーケンスで遅延評価
users.asSequence().filter { ... }.map { ... }.filter { ... }.first()

// 非推奨: ループ内でのリスト操作
var result = listOf<User>()
for (user in users) {
    if (user.isActive) {
        result = result + user  // 毎回新しいリストを生成
    }
}

// 推奨: 関数型スタイル
val result = users.filter { it.isActive }
```

## 式としての制御構文

Kotlin では`if`、`when`、`try`が式として値を返せる。これを活用して簡潔に書く。

```kotlin
// if式
val status = if (user.isActive) "Active" else "Inactive"

// when式
val message = when (result) {
    is Result.Success -> "Completed: ${result.value}"
    is Result.Failure -> "Failed: ${result.error.message}"
}

// when + sealed classで網羅性保証
val handler: suspend (UserEvent) -> Unit = when (event) {
    is UserEvent.Created -> ::handleCreated
    is UserEvent.Updated -> ::handleUpdated
    is UserEvent.Deleted -> ::handleDeleted
}

// try式（ただし、例外よりResultパターンを推奨）
val number = try {
    input.toInt()
} catch (e: NumberFormatException) {
    0
}
```

## 命名規則

### 関数名

```kotlin
// 動詞で始める
fun findUserById(id: UserId): User?
fun createOrder(request: OrderRequest): Order
fun validateEmail(email: String): Boolean

// Boolean を返す場合は is/has/can などで始める
fun isValid(): Boolean
fun hasPermission(user: User, action: Action): Boolean
fun canAccess(resource: Resource): Boolean

// 変換を表す場合は toXxx
fun User.toResponse(): UserResponse
fun String.toSlug(): String
```

### 変数名

```kotlin
// 意図が明確な名前
val activeUsers = users.filter { it.isActive }      // 良い
val list = users.filter { it.isActive }             // 悪い

// Boolean変数はis/has/shouldなどで始める
val isEnabled = config.featureFlags.enabled
val hasAccess = permissionService.check(user, resource)
val shouldRetry = retryCount < maxRetries
```
