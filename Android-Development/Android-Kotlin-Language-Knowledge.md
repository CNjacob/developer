# Android Kotlin 语法知识点

## 1. Kotlin 在 Android 中的定位

Kotlin 是现代 Android 开发的主流语言。Google 官方文档和 Jetpack 库大量使用 Kotlin 示例。

Kotlin 适合 Android 的原因：

- 语法比 Java 简洁。
- 类型系统支持空安全。
- `data class` 适合表达接口数据和 UI 状态。
- 扩展函数能增强 Android API 可读性。
- 协程适合处理网络、数据库、异步任务。
- 与 Java 互操作良好。

## 2. Hello World

```kotlin
fun main() {
    println("Hello Kotlin")
}
```

Android Activity：

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            Text("Hello Android Kotlin")
        }
    }
}
```

说明：

- `fun` 定义函数。
- `class` 定义类。
- Kotlin 通常不需要分号。
- Android 中 Activity 仍由系统启动，不从 `main` 进入。

## 3. 变量

Kotlin 有两种变量：

```kotlin
val name = "Tom"  // 只读引用，不能重新赋值
var age = 18      // 可变变量，可以重新赋值

age = 19
// name = "Jerry" // 编译错误
```

建议：

- 默认使用 `val`。
- 确实需要变化时再使用 `var`。

显式类型：

```kotlin
val userName: String = "Tom"
val count: Int = 10
val price: Double = 9.9
val enabled: Boolean = true
```

## 4. 基本类型

| 类型 | 说明 |
|------|------|
| `Int` | 整数 |
| `Long` | 长整数 |
| `Float` | 单精度浮点 |
| `Double` | 双精度浮点 |
| `Boolean` | 布尔 |
| `Char` | 字符 |
| `String` | 字符串 |

示例：

```kotlin
val id: Long = 10001L
val score: Float = 98.5f
val title: String = "Android"
```

字符串模板：

```kotlin
val name = "Tom"
val age = 18

val text = "name=$name, age=$age"
val nextYear = "next age=${age + 1}"
```

## 5. 空安全

Kotlin 默认类型不可为 null：

```kotlin
val name: String = "Tom"
// name = null // 编译错误
```

可空类型需要加 `?`：

```kotlin
var nickname: String? = null
```

安全调用：

```kotlin
val length = nickname?.length
```

Elvis 运算符：

```kotlin
val displayName = nickname ?: "Unknown"
```

非空断言：

```kotlin
val length = nickname!!.length
```

`!!` 可能导致崩溃，Android 开发中应尽量少用。

常见写法：

```kotlin
fun showName(name: String?) {
    if (name.isNullOrBlank()) {
        println("empty")
        return
    }

    // 经过判断后，Kotlin 能推断 name 非空
    println(name.length)
}
```

## 6. 函数

普通函数：

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

表达式函数：

```kotlin
fun add(a: Int, b: Int): Int = a + b
```

无返回值：

```kotlin
fun showMessage(message: String) {
    println(message)
}
```

默认参数：

```kotlin
fun request(url: String, timeout: Int = 10) {
    println("url=$url timeout=$timeout")
}

request("https://example.com")
request("https://example.com", timeout = 30)
```

命名参数：

```kotlin
request(
    url = "https://example.com",
    timeout = 20
)
```

## 7. 条件表达式

`if` 可以作为表达式：

```kotlin
val score = 85

val result = if (score >= 60) {
    "Pass"
} else {
    "Fail"
}
```

`when` 类似增强版 switch：

```kotlin
val tab = 1

val title = when (tab) {
    0 -> "Home"
    1 -> "Profile"
    2 -> "Settings"
    else -> "Unknown"
}
```

类型判断：

```kotlin
fun printValue(value: Any) {
    when (value) {
        is String -> println(value.length)
        is Int -> println(value + 1)
        else -> println("unknown")
    }
}
```

## 8. 循环

```kotlin
for (i in 0 until 5) {
    println(i) // 0,1,2,3,4
}
```

包含结尾：

```kotlin
for (i in 1..5) {
    println(i) // 1,2,3,4,5
}
```

倒序：

```kotlin
for (i in 5 downTo 1) {
    println(i)
}
```

步长：

```kotlin
for (i in 0..10 step 2) {
    println(i)
}
```

遍历集合：

```kotlin
val names = listOf("Tom", "Jerry")

for (name in names) {
    println(name)
}
```

带下标：

```kotlin
for ((index, name) in names.withIndex()) {
    println("$index: $name")
}
```

## 9. 类与对象

```kotlin
class User(
    val name: String,
    var age: Int
)
```

使用：

```kotlin
val user = User(name = "Tom", age = 18)
println(user.name)
user.age = 19
```

带方法：

```kotlin
class User(
    val name: String,
    val age: Int
) {
    fun displayText(): String {
        return "$name ($age)"
    }
}
```

## 10. data class

`data class` 适合表示数据。

```kotlin
data class User(
    val id: Long,
    val name: String,
    val age: Int
)
```

自动生成：

- `equals`
- `hashCode`
- `toString`
- `copy`
- `componentN`

示例：

```kotlin
val user = User(id = 1, name = "Tom", age = 18)

val older = user.copy(age = 19)

println(user)  // User(id=1, name=Tom, age=18)
```

Android 常见 UI 状态：

```kotlin
data class LoginUiState(
    val loading: Boolean = false,
    val username: String = "",
    val errorMessage: String? = null
)
```

## 11. object 与 companion object

单例：

```kotlin
object AppConfig {
    const val BASE_URL = "https://api.example.com/"
}
```

使用：

```kotlin
val url = AppConfig.BASE_URL
```

伴生对象：

```kotlin
class DetailActivity : AppCompatActivity() {
    companion object {
        const val EXTRA_USER_ID = "extra_user_id"

        fun createIntent(context: Context, userId: Long): Intent {
            return Intent(context, DetailActivity::class.java).apply {
                putExtra(EXTRA_USER_ID, userId)
            }
        }
    }
}
```

调用：

```kotlin
val intent = DetailActivity.createIntent(this, 1001)
startActivity(intent)
```

## 12. 继承与接口

Kotlin 类默认不能被继承，需要 `open`。

```kotlin
open class Animal {
    open fun eat() {
        println("eat")
    }
}

class Dog : Animal() {
    override fun eat() {
        println("dog eat")
    }
}
```

接口：

```kotlin
interface OnLoginListener {
    fun onSuccess(token: String)
    fun onError(error: Throwable)
}
```

实现：

```kotlin
class LoginLogger : OnLoginListener {
    override fun onSuccess(token: String) {
        println("token=$token")
    }

    override fun onError(error: Throwable) {
        println(error.message)
    }
}
```

## 13. 可见性

| 修饰符 | 说明 |
|--------|------|
| `public` | 默认，任意位置可见 |
| `internal` | 当前模块可见 |
| `protected` | 子类可见 |
| `private` | 当前文件或当前类可见 |

示例：

```kotlin
internal class UserRepository {
    private val cache = mutableMapOf<Long, User>()
}
```

## 14. 集合

不可变 List：

```kotlin
val names = listOf("Tom", "Jerry")
```

可变 List：

```kotlin
val names = mutableListOf<String>()
names.add("Tom")
names.add("Jerry")
```

Set：

```kotlin
val tags = setOf("android", "kotlin")
```

Map：

```kotlin
val scores = mapOf(
    "Tom" to 90,
    "Jerry" to 80
)

val tomScore = scores["Tom"]
```

常用集合操作：

```kotlin
data class User(val id: Long, val name: String, val age: Int)

val users = listOf(
    User(1, "Tom", 18),
    User(2, "Jerry", 20)
)

val adultNames = users
    .filter { it.age >= 18 }    // 过滤
    .map { it.name }            // 转换

val first = users.firstOrNull()
val byId = users.associateBy { it.id }
```

## 15. Lambda

Lambda 是匿名函数。

```kotlin
val add: (Int, Int) -> Int = { a, b ->
    a + b
}

println(add(1, 2))
```

Android 点击事件：

```kotlin
button.setOnClickListener {
    Toast.makeText(this, "clicked", Toast.LENGTH_SHORT).show()
}
```

高阶函数：

```kotlin
fun measure(block: () -> Unit) {
    val start = System.currentTimeMillis()
    block()
    val cost = System.currentTimeMillis() - start
    println("cost=$cost")
}

measure {
    println("do work")
}
```

## 16. 扩展函数

扩展函数可以给已有类型添加函数。

```kotlin
fun String.isValidPhone(): Boolean {
    return this.length == 11 && this.all { it.isDigit() }
}
```

使用：

```kotlin
val valid = "13800138000".isValidPhone()
```

Android 示例：

```kotlin
fun View.visible() {
    visibility = View.VISIBLE
}

fun View.gone() {
    visibility = View.GONE
}
```

使用：

```kotlin
progressBar.visible()
contentView.gone()
```

## 17. 作用域函数

Kotlin 常见作用域函数：

| 函数 | 返回值 | 适合场景 |
|------|--------|----------|
| `let` | lambda 结果 | 空值判断、转换 |
| `run` | lambda 结果 | 计算一段逻辑 |
| `with` | lambda 结果 | 对同一对象做多次操作 |
| `apply` | 对象本身 | 初始化对象 |
| `also` | 对象本身 | 附加操作、日志 |

`let`：

```kotlin
val name: String? = "Tom"

name?.let {
    println(it.length)
}
```

`apply`：

```kotlin
val intent = Intent(this, DetailActivity::class.java).apply {
    putExtra("user_id", 1001)
    putExtra("from", "home")
}
```

`also`：

```kotlin
val user = loadUser().also {
    Log.d("User", "loaded: $it")
}
```

## 18. sealed class

`sealed class` 适合表达有限状态。

```kotlin
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: String) : UiState()
    data class Error(val message: String) : UiState()
}
```

使用：

```kotlin
fun render(state: UiState) {
    when (state) {
        UiState.Loading -> println("loading")
        is UiState.Success -> println(state.data)
        is UiState.Error -> println(state.message)
    }
}
```

好处：

- 状态类型清晰。
- `when` 可以做穷尽检查。
- UI 状态建模更安全。

## 19. 泛型

```kotlin
class ApiResult<T>(
    val data: T?,
    val message: String
)
```

使用：

```kotlin
val result: ApiResult<User> = ApiResult(
    data = User(1, "Tom", 18),
    message = "ok"
)
```

泛型函数：

```kotlin
fun <T> firstOrNull(list: List<T>): T? {
    return if (list.isEmpty()) null else list[0]
}
```

泛型约束：

```kotlin
fun <T : Number> doubleValue(value: T): Double {
    return value.toDouble() * 2
}
```

## 20. 异常处理

```kotlin
try {
    val result = 10 / 0
    println(result)
} catch (e: ArithmeticException) {
    Log.e("Demo", "calculate failed", e)
} finally {
    Log.d("Demo", "always run")
}
```

`runCatching`：

```kotlin
val result = runCatching {
    repository.loadUser(1)
}

result.onSuccess { user ->
    println(user.name)
}.onFailure { error ->
    Log.e("User", "load failed", error)
}
```

## 21. 协程基础

协程用于异步任务，尤其适合网络和数据库。

ViewModel 示例：

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    fun loadUser(id: Long) {
        viewModelScope.launch {
            // viewModelScope 会在 ViewModel 清理时自动取消
            val user = repository.loadUser(id)
            Log.d("User", user.name)
        }
    }
}
```

切换线程：

```kotlin
suspend fun loadUser(id: Long): User {
    return withContext(Dispatchers.IO) {
        // IO 线程执行网络或数据库
        api.getUser(id)
    }
}
```

常用 Dispatcher：

| Dispatcher | 用途 |
|------------|------|
| `Dispatchers.Main` | UI 线程 |
| `Dispatchers.IO` | 网络、数据库、文件 |
| `Dispatchers.Default` | CPU 密集计算 |

## 22. Flow

Flow 表示异步数据流。

```kotlin
class UserRepository {
    fun observeUserName(): Flow<String> = flow {
        emit("Tom")
        delay(1000)
        emit("Jerry")
    }
}
```

ViewModel 中收集：

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _name = MutableStateFlow("")
    val name: StateFlow<String> = _name

    init {
        viewModelScope.launch {
            repository.observeUserName().collect { value ->
                _name.value = value
            }
        }
    }
}
```

Compose 中使用：

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val name by viewModel.name.collectAsState()

    Text(text = name)
}
```

## 23. Android 常见 Kotlin 写法

### 23.1 Intent 创建

```kotlin
class DetailActivity : AppCompatActivity() {
    companion object {
        private const val EXTRA_USER_ID = "extra_user_id"

        fun createIntent(context: Context, userId: Long): Intent {
            return Intent(context, DetailActivity::class.java).apply {
                putExtra(EXTRA_USER_ID, userId)
            }
        }
    }
}
```

### 23.2 ViewModel UI 状态

```kotlin
data class LoginUiState(
    val loading: Boolean = false,
    val username: String = "",
    val password: String = "",
    val error: String? = null
)
```

```kotlin
class LoginViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(LoginUiState())
    val uiState: StateFlow<LoginUiState> = _uiState

    fun onUsernameChanged(value: String) {
        _uiState.value = _uiState.value.copy(username = value)
    }

    fun onPasswordChanged(value: String) {
        _uiState.value = _uiState.value.copy(password = value)
    }
}
```

### 23.3 Repository

```kotlin
class UserRepository(
    private val api: UserApi,
    private val dao: UserDao
) {
    suspend fun refreshUser(id: Long): User {
        val dto = api.getUser(id)
        val entity = UserEntity(
            id = dto.id,
            name = dto.name
        )

        dao.save(entity)

        return User(
            id = entity.id,
            name = entity.name
        )
    }
}
```

## 24. Kotlin 与 Java 互操作

Kotlin 调 Java：

```kotlin
val user = JavaUser("Tom", 18)
println(user.name)
```

Java 调 Kotlin：

```kotlin
class KotlinUtils {
    companion object {
        @JvmStatic
        fun formatName(name: String): String {
            return name.trim()
        }
    }
}
```

Java 中：

```java
String name = KotlinUtils.formatName(" Tom ");
```

注意平台类型：

```kotlin
val text = javaApi.getText()
```

Java 返回值在 Kotlin 中可能被看作平台类型，空安全不完全由编译器保证。调用 Java API 时仍要注意 null。

## 25. 常见误区

| 误区 | 正确理解 |
|------|----------|
| `val` 表示对象不可变 | `val` 只是引用不可重新赋值，对象内部可能仍可变 |
| Kotlin 不会空指针 | 使用 `!!`、Java 平台类型仍可能空指针 |
| 协程就是线程 | 协程是并发抽象，不等于线程 |
| `GlobalScope` 很方便 | Android 中容易泄漏，不建议随意使用 |
| `data class` 可用于所有类 | 它适合纯数据，不适合复杂行为对象 |
| Compose 状态随便写局部变量 | UI 状态应使用 `remember`、ViewModel 或状态容器 |

## 26. 学习顺序

```text
val / var / 函数
  |
  v
空安全
  |
  v
类 / data class / object
  |
  v
集合 / lambda / 扩展函数
  |
  v
sealed class / 泛型
  |
  v
协程 / Flow
  |
  v
Android ViewModel / Compose 实战
```

## 27. 参考资料

- Kotlin Docs：https://kotlinlang.org/docs/home.html
- Kotlin Basic syntax：https://kotlinlang.org/docs/basic-syntax.html
- Kotlin Null safety：https://kotlinlang.org/docs/null-safety.html
- Kotlin Coroutines：https://kotlinlang.org/docs/coroutines-overview.html
- Android Kotlin：https://developer.android.com/kotlin

## 28. 总结

Android Kotlin 最重要的知识点：

| 主题 | 价值 |
|------|------|
| 空安全 | 降低空指针崩溃 |
| data class | 简化数据和 UI 状态建模 |
| Lambda | 简化回调和集合处理 |
| 扩展函数 | 提升 Android API 可读性 |
| sealed class | 安全表达有限状态 |
| 协程和 Flow | 处理异步、数据流和 UI 状态 |

学 Kotlin 不要只背语法。更好的方式是把语法放进 Android 场景：页面状态、点击事件、网络请求、数据库读取、错误处理和 Compose UI。
