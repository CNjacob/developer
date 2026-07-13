# Android 基础知识与常用开发技术

## 1. Android 是什么

Android 是一个面向移动设备的操作系统和应用平台。Android App 通常运行在手机、平板、折叠屏、电视、手表、车机等设备上。

Android App 的核心特点：

- 运行在受系统管理的进程中。
- 通过组件和系统交互。
- UI、后台任务、权限、存储都受系统生命周期约束。
- 应用需要适配不同屏幕、系统版本、语言和权限策略。

## 2. Android App 基本结构

```text
App
  |
  |-- Manifest: 声明应用信息、组件、权限
  |-- Activity: 页面
  |-- Service: 后台服务
  |-- BroadcastReceiver: 广播接收
  |-- ContentProvider: 跨应用数据提供
  |-- Resources: 图片、字符串、颜色、布局
  |-- Gradle: 构建和依赖
```

最常见的新手项目只会先用到：

- `Activity`
- `ViewModel`
- Compose 或 XML 布局
- 网络请求
- 本地存储
- 权限

## 3. AndroidManifest.xml

`AndroidManifest.xml` 是 App 清单文件。

示例：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <!-- 声明网络权限，否则不能访问网络 -->
    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:theme="@style/Theme.MyApp"
        android:label="@string/app_name"
        android:allowBackup="true">

        <!-- Activity 是一个页面入口 -->
        <activity
            android:name=".MainActivity"
            android:exported="true">

            <!-- MAIN + LAUNCHER 表示桌面点击图标启动这个 Activity -->
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

关键点：

| 配置 | 作用 |
|------|------|
| `uses-permission` | 声明权限 |
| `application` | 应用级配置 |
| `activity` | 声明页面 |
| `exported` | 是否允许其他 App 启动该组件 |
| `intent-filter` | 声明组件能响应哪些 Intent |

## 4. 四大组件

### 4.1 Activity

Activity 代表一个可见页面。

Kotlin 示例：

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // 设置页面内容
        setContentView(R.layout.activity_main)
    }
}
```

Java 示例：

```java
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // 设置页面内容
        setContentView(R.layout.activity_main);
    }
}
```

### 4.2 Service

Service 用于执行没有界面的后台任务。现代 Android 中，长时间后台任务应优先考虑 `WorkManager` 或前台服务。

示例：

```kotlin
class MusicService : Service() {
    override fun onBind(intent: Intent?): IBinder? {
        // 不提供绑定能力时返回 null
        return null
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // 这里执行后台逻辑，例如播放音乐
        return START_STICKY
    }
}
```

### 4.3 BroadcastReceiver

BroadcastReceiver 用于接收广播，例如网络变化、系统事件、自定义事件。

```kotlin
class BootReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // onReceive 执行时间要短，不适合做耗时任务
        Log.d("BootReceiver", "receive action: ${intent.action}")
    }
}
```

### 4.4 ContentProvider

ContentProvider 用于跨应用共享结构化数据。普通业务 App 较少手写，更多会用到系统 Provider，例如相册、联系人。

## 5. Activity 生命周期

```text
onCreate
   |
   v
onStart
   |
   v
onResume  <---- 用户正在交互
   |
   v
onPause
   |
   v
onStop
   |
   v
onDestroy
```

示例：

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        Log.d("Life", "onCreate: 初始化页面和数据")
    }

    override fun onStart() {
        super.onStart()
        Log.d("Life", "onStart: 页面即将可见")
    }

    override fun onResume() {
        super.onResume()
        Log.d("Life", "onResume: 页面可交互")
    }

    override fun onPause() {
        super.onPause()
        Log.d("Life", "onPause: 页面失去焦点")
    }

    override fun onStop() {
        super.onStop()
        Log.d("Life", "onStop: 页面不可见")
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.d("Life", "onDestroy: 页面销毁")
    }
}
```

常见使用：

| 方法 | 适合做什么 |
|------|------------|
| `onCreate` | 初始化 UI、绑定 ViewModel |
| `onStart` | 注册可见期间需要的监听 |
| `onResume` | 开始动画、刷新轻量数据 |
| `onPause` | 暂停动画、保存草稿 |
| `onStop` | 释放可见期间资源 |
| `onDestroy` | 最终清理，不能依赖一定被调用 |

## 6. Intent

Intent 是组件之间通信的消息。

### 6.1 显式 Intent

用于启动自己 App 内明确的 Activity。

```kotlin
val intent = Intent(this, DetailActivity::class.java)
intent.putExtra("user_id", 1001)
startActivity(intent)
```

接收参数：

```kotlin
class DetailActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val userId = intent.getIntExtra("user_id", -1)
        Log.d("Detail", "userId=$userId")
    }
}
```

### 6.2 隐式 Intent

用于交给系统找合适 App 处理。

```kotlin
val intent = Intent(Intent.ACTION_VIEW).apply {
    data = Uri.parse("https://developer.android.com")
}
startActivity(intent)
```

## 7. UI 开发路线

Android UI 有两条主线：

| 方式 | 特点 |
|------|------|
| Jetpack Compose | 现代声明式 UI，Kotlin 优先 |
| XML View | 传统 UI，老项目大量使用 |

建议：

- 新项目优先学习 Compose。
- 想维护老项目必须理解 XML View。
- 两者可以混用。

## 8. Jetpack Compose 基础

Compose 使用函数声明界面。

```kotlin
@Composable
fun Greeting(name: String) {
    // Text 是一个基础文本组件
    Text(text = "Hello $name")
}
```

带状态的示例：

```kotlin
@Composable
fun CounterScreen() {
    // remember 保存组合期间的状态
    var count by remember { mutableStateOf(0) }

    Column {
        Text(text = "Count: $count")

        Button(onClick = {
            // 修改状态后，Compose 会重新绘制依赖该状态的 UI
            count++
        }) {
            Text("Add")
        }
    }
}
```

常见布局：

```kotlin
@Composable
fun UserCard(name: String, age: Int) {
    Column(
        modifier = Modifier
            .fillMaxWidth()
            .padding(16.dp)
    ) {
        Text(text = name, style = MaterialTheme.typography.titleLarge)
        Text(text = "Age: $age")
    }
}
```

核心概念：

| 概念 | 说明 |
|------|------|
| `@Composable` | 声明一个 UI 函数 |
| `remember` | 在组合期间保存状态 |
| `mutableStateOf` | 可观察状态，变化后触发重组 |
| `Modifier` | 控制大小、间距、点击、背景等 |
| `Column/Row/Box` | 垂直、水平、层叠布局 |

## 9. XML View 基础

`activity_main.xml`：

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:id="@+id/titleText"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello Android"
        android:textSize="24sp" />

    <Button
        android:id="@+id/addButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Add" />
</LinearLayout>
```

Kotlin 绑定：

```kotlin
class MainActivity : AppCompatActivity() {
    private var count = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val titleText = findViewById<TextView>(R.id.titleText)
        val addButton = findViewById<Button>(R.id.addButton)

        addButton.setOnClickListener {
            count++
            titleText.text = "Count: $count"
        }
    }
}
```

## 10. 资源系统

常见资源目录：

| 目录 | 内容 |
|------|------|
| `res/drawable` | 图片、shape、vector |
| `res/mipmap` | App 图标 |
| `res/layout` | XML 布局 |
| `res/values/strings.xml` | 字符串 |
| `res/values/colors.xml` | 颜色 |
| `res/values/themes.xml` | 主题 |

`strings.xml`：

```xml
<resources>
    <string name="app_name">My App</string>
    <string name="login">登录</string>
</resources>
```

代码中使用：

```kotlin
val appName = getString(R.string.app_name)
```

好处：

- 便于多语言。
- 避免硬编码。
- 统一管理文本和样式。

## 11. 权限

Android 权限分为普通权限和危险权限。危险权限需要运行时申请。

清单声明：

```xml
<uses-permission android:name="android.permission.CAMERA" />
```

Compose / Activity 中使用 Activity Result API：

```kotlin
class MainActivity : ComponentActivity() {
    private val requestCamera = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            Log.d("Permission", "camera granted")
        } else {
            Log.d("Permission", "camera denied")
        }
    }

    fun askCameraPermission() {
        requestCamera.launch(Manifest.permission.CAMERA)
    }
}
```

权限处理原则：

- 用到时再申请。
- 申请前解释用途。
- 被拒绝后给降级方案。
- 不要申请用不到的权限。

## 12. 网络请求

Android 常用网络库：

| 库 | 作用 |
|----|------|
| OkHttp | HTTP 客户端 |
| Retrofit | API 接口声明和请求封装 |
| kotlinx.serialization / Gson / Moshi | JSON 序列化 |

Retrofit 示例：

```kotlin
data class UserDto(
    val id: Long,
    val name: String
)

interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): UserDto
}
```

创建 Retrofit：

```kotlin
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(GsonConverterFactory.create())
    .build()

val api = retrofit.create(UserApi::class.java)
```

在协程中请求：

```kotlin
class UserRepository(private val api: UserApi) {
    suspend fun loadUser(id: Long): UserDto {
        // suspend 函数不会阻塞线程，但需要在协程中调用
        return api.getUser(id)
    }
}
```

注意：

- 网络请求不能在主线程直接阻塞执行。
- 需要处理超时、失败、空数据、错误码。
- API 返回对象和 UI 模型最好分开。

## 13. 本地存储

### 13.1 SharedPreferences

适合存少量简单配置。

```kotlin
val prefs = getSharedPreferences("settings", Context.MODE_PRIVATE)

// 保存
prefs.edit()
    .putBoolean("dark_mode", true)
    .apply()

// 读取
val darkMode = prefs.getBoolean("dark_mode", false)
```

### 13.2 DataStore

DataStore 是更现代的键值存储方案，适合替代 SharedPreferences。

### 13.3 Room

Room 是 SQLite 的封装，适合结构化数据。

Entity：

```kotlin
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: Long,
    val name: String,
    val age: Int
)
```

DAO：

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :id")
    suspend fun findById(id: Long): UserEntity?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun save(user: UserEntity)
}
```

Database：

```kotlin
@Database(
    entities = [UserEntity::class],
    version = 1
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

## 14. 线程与协程

Android 主线程负责 UI，耗时任务不能阻塞主线程。

常见线程选择：

| 任务 | 建议 |
|------|------|
| 更新 UI | 主线程 |
| 网络请求 | IO 线程 / 协程 |
| 数据库读写 | IO 线程 / 协程 |
| CPU 密集计算 | Default 线程池 |

ViewModel 中使用协程：

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _name = MutableStateFlow("")
    val name: StateFlow<String> = _name

    fun loadUser(id: Long) {
        viewModelScope.launch {
            try {
                val user = repository.loadUser(id)
                _name.value = user.name
            } catch (e: Exception) {
                Log.e("UserViewModel", "load failed", e)
            }
        }
    }
}
```

## 15. 推荐架构

初学阶段可以使用简化分层：

```text
UI(Activity/Compose)
        |
        v
ViewModel
        |
        v
Repository
        |
        v
Remote API / Local Database
```

职责：

| 层 | 职责 |
|----|------|
| UI | 展示状态，接收用户操作 |
| ViewModel | 保存 UI 状态，处理页面逻辑 |
| Repository | 封装数据来源 |
| API / DB | 具体网络和本地数据访问 |

UI 状态示例：

```kotlin
data class UserUiState(
    val loading: Boolean = false,
    val name: String = "",
    val errorMessage: String? = null
)
```

ViewModel：

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState

    fun loadUser(id: Long) {
        viewModelScope.launch {
            _uiState.value = UserUiState(loading = true)

            runCatching {
                repository.loadUser(id)
            }.onSuccess { user ->
                _uiState.value = UserUiState(name = user.name)
            }.onFailure { error ->
                _uiState.value = UserUiState(errorMessage = error.message)
            }
        }
    }
}
```

Compose 页面：

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val state by viewModel.uiState.collectAsState()

    when {
        state.loading -> CircularProgressIndicator()
        state.errorMessage != null -> Text("Error: ${state.errorMessage}")
        else -> Text("Name: ${state.name}")
    }
}
```

## 16. Navigation

Navigation 用于页面跳转和参数传递。

Compose Navigation 示例：

```kotlin
@Composable
fun AppNavHost() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onOpenDetail = { userId ->
                    navController.navigate("detail/$userId")
                }
            )
        }

        composable("detail/{userId}") { backStackEntry ->
            val userId = backStackEntry.arguments
                ?.getString("userId")
                ?.toLongOrNull()

            DetailScreen(userId = userId)
        }
    }
}
```

## 17. 图片加载

常用库：

| 库 | 特点 |
|----|------|
| Coil | Kotlin 友好，Compose 支持好 |
| Glide | 老牌稳定，传统 View 项目常见 |

Compose + Coil 示例：

```kotlin
@Composable
fun Avatar(url: String) {
    AsyncImage(
        model = url,
        contentDescription = "avatar",
        modifier = Modifier.size(64.dp)
    )
}
```

## 18. 后台任务

| 场景 | 建议 |
|------|------|
| 可延迟、保证执行 | WorkManager |
| 用户能感知的持续任务 | Foreground Service |
| 页面内短任务 | viewModelScope / lifecycleScope |
| 精准定时 | AlarmManager |

WorkManager 示例：

```kotlin
class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            // 执行同步任务
            Result.success()
        } catch (e: Exception) {
            // 告诉系统稍后重试
            Result.retry()
        }
    }
}
```

启动任务：

```kotlin
val request = OneTimeWorkRequestBuilder<SyncWorker>().build()
WorkManager.getInstance(context).enqueue(request)
```

## 19. 测试

Android 测试分为：

| 类型 | 运行位置 | 用途 |
|------|----------|------|
| Local Unit Test | JVM | 测普通 Kotlin/Java 逻辑 |
| Instrumented Test | 设备/模拟器 | 测 Android 相关逻辑 |
| UI Test | 设备/模拟器 | 测页面交互 |

单元测试示例：

```kotlin
class PriceCalculator {
    fun total(price: Int, count: Int): Int {
        return price * count
    }
}
```

```kotlin
class PriceCalculatorTest {
    @Test
    fun total_returnsPriceMultiplyCount() {
        val calculator = PriceCalculator()

        val result = calculator.total(price = 10, count = 3)

        assertEquals(30, result)
    }
}
```

## 20. 调试与性能

常用工具：

| 工具 | 用途 |
|------|------|
| Logcat | 看日志 |
| Debugger | 断点调试 |
| Layout Inspector | 看 UI 层级 |
| Profiler | 分析 CPU、内存、网络 |
| APK Analyzer | 分析包体积 |
| Lint | 静态检查 |

常见性能原则：

- 不要在主线程做网络和数据库重操作。
- 列表要复用或使用 LazyColumn。
- 图片要压缩和缓存。
- 避免内存泄漏，长生命周期对象不要持有 Activity。
- 页面状态放 ViewModel，不要全塞 Activity。

## 21. 发布基础

Android 发布产物：

| 产物 | 说明 |
|------|------|
| APK | 可直接安装 |
| AAB | Android App Bundle，Google Play 推荐发布格式 |

构建命令：

```bash
# Debug APK
./gradlew assembleDebug

# Release APK
./gradlew assembleRelease

# Release AAB
./gradlew bundleRelease
```

发布前检查：

- `versionCode` 是否递增。
- `versionName` 是否正确。
- Release 签名是否配置。
- 混淆规则是否正确。
- 隐私政策、权限说明是否符合要求。
- 真机和不同屏幕尺寸是否测试。

## 22. 新手常见误区

| 误区 | 正确理解 |
|------|----------|
| Activity 里写所有逻辑 | 页面逻辑应逐步放到 ViewModel / Repository |
| 网络请求直接写在主线程 | 使用协程或异步 API |
| 权限在安装时一次申请 | 危险权限需要运行时申请 |
| 只测一台手机 | Android 碎片化，需要多尺寸、多版本验证 |
| 只会照着 UI 写代码 | 还要理解生命周期、状态、数据流 |
| 看到 Compose 就不用学 XML | 老项目仍有大量 XML View |

## 23. 学习路线

```text
Kotlin / Java 基础
        |
        v
Activity + 生命周期 + Intent
        |
        v
UI: Compose + XML 基础
        |
        v
ViewModel + State + Navigation
        |
        v
网络 + 本地存储 + 权限
        |
        v
测试 + 调试 + 性能
        |
        v
打包 + 签名 + 发布
```

## 24. 参考资料

- Android App fundamentals：https://developer.android.com/guide/components/fundamentals
- Jetpack Compose：https://developer.android.com/develop/ui/compose/documentation
- App architecture：https://developer.android.com/topic/architecture
- Android permissions：https://developer.android.com/guide/topics/permissions/overview
- WorkManager：https://developer.android.com/topic/libraries/architecture/workmanager

## 25. 总结

Android 学习要抓住三条主线：

| 主线 | 内容 |
|------|------|
| 组件生命周期 | Activity、Service、Receiver、Provider |
| UI 与状态 | Compose/XML、ViewModel、StateFlow |
| 数据与系统能力 | 网络、数据库、权限、后台任务、调试发布 |

初学阶段优先做小闭环：一个页面、一个按钮、一次跳转、一次网络请求、一次本地保存。每个闭环都跑通，比一开始背大量 API 更有效。
