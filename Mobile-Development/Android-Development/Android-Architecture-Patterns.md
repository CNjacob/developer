# Android 常见架构类型、目录结构与示例代码

## 1. 为什么要学习架构

Android 初学者常见的问题不是“不会写一个按钮点击”，而是项目变大后不知道代码该放哪里：

- 网络请求写在 Activity 里，页面越来越胖。
- 数据库、缓存、接口、UI 状态混在一起。
- 改一个字段影响很多页面。
- 代码难测试，Bug 只能靠手点页面验证。
- 多人协作时目录混乱，互相冲突。

架构的目标不是让代码变复杂，而是让代码职责清晰、依赖方向稳定、方便测试和维护。

## 2. 官方推荐的基础分层

现代 Android 官方架构建议通常围绕这些层组织：

```text
UI Layer
  |
  v
Domain Layer，可选
  |
  v
Data Layer
```

| 层 | 常见类 | 职责 |
|----|--------|------|
| UI Layer | Activity、Fragment、Composable、ViewModel、UiState | 展示界面、接收用户事件、持有 UI 状态 |
| Domain Layer | UseCase、Interactor | 封装可复用业务规则，可选 |
| Data Layer | Repository、DataSource、Api、Dao、Entity、Dto | 提供数据，处理网络、本地数据库、缓存 |

依赖方向建议：

```text
UI -> Domain -> Data
```

或者无 Domain 时：

```text
UI -> Data
```

注意：这里的箭头表示“上层调用下层”。实际工程中如果使用接口反转，Domain 可以只依赖抽象接口，不直接依赖 Data 实现。

## 3. 架构选型速查

| 架构 | 适合场景 | 新手建议 |
|------|----------|----------|
| 无架构 / 简单分层 | Demo、极小页面、一次性工具 | 可以入门，但不要用于长期项目 |
| MVC | 老项目、传统 View 项目 | 了解即可，Activity 容易变胖 |
| MVP | 老项目、Java/XML 项目、需要隔离 View | 能看懂维护即可 |
| MVVM | 大多数现代 Android 项目 | 推荐重点学习 |
| MVI / UDF | 状态复杂、Compose、强一致 UI 状态 | 有 MVVM 基础后学习 |
| Clean Architecture | 中大型业务、复杂规则、高测试要求 | 不要一开始就过度套模板 |
| 模块化架构 | 多团队、多业务线、构建变慢 | 中大型项目再引入 |

## 4. 示例业务：用户列表

下面所有架构都用同一个业务示例：

```text
页面打开
  |
  v
加载用户列表
  |
  v
展示 loading
  |
  v
成功显示列表 / 失败显示错误
```

数据模型：

```kotlin
data class User(
    val id: Long,
    val name: String,
    val age: Int
)
```

为了方便阅读，示例代码会省略部分 import 和 Gradle 依赖。

## 5. 无架构 / Activity 直写

### 5.1 目录结构

```text
app/
  src/main/java/com/example/app/
    MainActivity.kt
    User.kt
```

### 5.2 示例代码

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            var loading by remember { mutableStateOf(false) }
            var users by remember { mutableStateOf<List<User>>(emptyList()) }
            var error by remember { mutableStateOf<String?>(null) }

            LaunchedEffect(Unit) {
                loading = true
                error = null

                try {
                    // 示例：真实项目不要把网络请求直接写在 Activity 中
                    delay(1000)
                    users = listOf(
                        User(1, "Tom", 18),
                        User(2, "Jerry", 20)
                    )
                } catch (e: Exception) {
                    error = e.message
                } finally {
                    loading = false
                }
            }

            when {
                loading -> Text("Loading...")
                error != null -> Text("Error: $error")
                else -> Column {
                    users.forEach { user ->
                        Text("${user.name}, ${user.age}")
                    }
                }
            }
        }
    }
}
```

### 5.3 优缺点

优点：

- 上手最快。
- Demo 代码少。

缺点：

- UI、数据、业务逻辑混在一起。
- Activity / Composable 很快变大。
- 难测试。
- 数据逻辑无法复用。

适合：学习 API、写临时 Demo。正式项目不推荐长期使用。

## 6. MVC

MVC 是 Model、View、Controller。

在 Android 老项目中经常这样对应：

| MVC 角色 | Android 中常见承担者 |
|----------|----------------------|
| Model | 数据类、Repository、数据库、网络 |
| View | XML、View、Activity/Fragment 的显示部分 |
| Controller | Activity/Fragment |

问题是 Android 的 Activity/Fragment 天然既处理生命周期，又控制页面，又绑定 View，很容易变成“巨型 Controller”。

### 6.1 目录结构

```text
app/src/main/java/com/example/app/
  model/
    User.kt
    UserRepository.kt
  controller/
    UserActivity.kt
  view/
    UserAdapter.kt
```

### 6.2 Model

```kotlin
data class User(
    val id: Long,
    val name: String,
    val age: Int
)
```

```kotlin
class UserRepository {
    suspend fun getUsers(): List<User> {
        // 这里模拟网络或数据库请求
        delay(500)

        return listOf(
            User(1, "Tom", 18),
            User(2, "Jerry", 20)
        )
    }
}
```

### 6.3 Controller

```kotlin
class UserActivity : AppCompatActivity() {
    private val repository = UserRepository()

    private lateinit var titleView: TextView
    private lateinit var contentView: TextView
    private lateinit var progressBar: ProgressBar

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user)

        titleView = findViewById(R.id.titleView)
        contentView = findViewById(R.id.contentView)
        progressBar = findViewById(R.id.progressBar)

        loadUsers()
    }

    private fun loadUsers() {
        lifecycleScope.launch {
            progressBar.visibility = View.VISIBLE
            contentView.text = ""

            try {
                // Controller 直接调用 Model
                val users = repository.getUsers()

                // Controller 同时负责更新 View
                titleView.text = "User List"
                contentView.text = users.joinToString("\n") {
                    "${it.name}, ${it.age}"
                }
            } catch (e: Exception) {
                contentView.text = "Load failed: ${e.message}"
            } finally {
                progressBar.visibility = View.GONE
            }
        }
    }
}
```

### 6.4 优缺点

优点：

- 比完全无架构稍清晰。
- Model 可以复用。
- 老项目容易遇到。

缺点：

- Activity 仍然承担太多职责。
- Controller 和 Android UI 强绑定，测试困难。
- 页面复杂后维护成本高。

适合：简单老项目、学习历史架构。新项目更推荐 MVVM 或 MVI。

## 7. MVP

MVP 是 Model、View、Presenter。

核心思想：

```text
View 只负责显示和用户事件
Presenter 负责页面逻辑
Model 负责数据
```

Android 中：

| MVP 角色 | 常见类 |
|----------|--------|
| Model | Repository、DataSource |
| View | Activity/Fragment + View 接口 |
| Presenter | XxxPresenter |

### 7.1 目录结构

```text
app/src/main/java/com/example/app/user/
  model/
    User.kt
    UserRepository.kt
  mvp/
    UserContract.kt
    UserPresenter.kt
    UserActivity.kt
```

### 7.2 Contract

```kotlin
interface UserContract {
    interface View {
        // 显示加载状态
        fun showLoading()

        // 隐藏加载状态
        fun hideLoading()

        // 展示用户列表
        fun showUsers(users: List<User>)

        // 展示错误信息
        fun showError(message: String)
    }

    interface Presenter {
        // 页面创建后调用
        fun attach(view: View)

        // 页面销毁时调用，避免 Presenter 持有 Activity 导致泄漏
        fun detach()

        // 加载用户列表
        fun loadUsers()
    }
}
```

### 7.3 Presenter

```kotlin
class UserPresenter(
    private val repository: UserRepository
) : UserContract.Presenter {

    private var view: UserContract.View? = null
    private var job: Job? = null

    override fun attach(view: UserContract.View) {
        this.view = view
    }

    override fun detach() {
        // Activity 销毁后释放 View 引用，避免内存泄漏
        view = null

        // 取消还没完成的异步任务
        job?.cancel()
    }

    override fun loadUsers() {
        view?.showLoading()

        job = CoroutineScope(Dispatchers.Main).launch {
            try {
                val users = withContext(Dispatchers.IO) {
                    repository.getUsers()
                }

                view?.showUsers(users)
            } catch (e: Exception) {
                view?.showError(e.message ?: "Unknown error")
            } finally {
                view?.hideLoading()
            }
        }
    }
}
```

### 7.4 View

```kotlin
class UserActivity : AppCompatActivity(), UserContract.View {
    private val presenter = UserPresenter(UserRepository())

    private lateinit var progressBar: ProgressBar
    private lateinit var contentView: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_user)

        progressBar = findViewById(R.id.progressBar)
        contentView = findViewById(R.id.contentView)

        presenter.attach(this)
        presenter.loadUsers()
    }

    override fun onDestroy() {
        super.onDestroy()
        presenter.detach()
    }

    override fun showLoading() {
        progressBar.visibility = View.VISIBLE
    }

    override fun hideLoading() {
        progressBar.visibility = View.GONE
    }

    override fun showUsers(users: List<User>) {
        contentView.text = users.joinToString("\n") {
            "${it.name}, ${it.age}"
        }
    }

    override fun showError(message: String) {
        contentView.text = message
    }
}
```

### 7.5 优缺点

优点：

- Activity 变薄，逻辑在 Presenter。
- Presenter 可以单元测试。
- 适合 Java + XML 老项目。

缺点：

- View 接口方法多时会膨胀。
- Presenter 需要手动处理生命周期和取消任务。
- 状态分散在多个 `showXxx()` 方法里，复杂页面容易不一致。

适合：老项目维护、传统 View 体系。新 Kotlin 项目通常更推荐 ViewModel + State。

## 8. MVVM

MVVM 是 Model、View、ViewModel。

现代 Android 最常见的架构之一。

```text
View(Activity/Fragment/Composable)
        |
        | user event
        v
ViewModel
        |
        | calls
        v
Repository
        |
        v
Api / Dao / DataSource
```

核心思想：

- View 只负责展示状态和发送事件。
- ViewModel 持有 UI 状态。
- Repository 提供数据。
- View 观察状态变化并自动刷新。

### 8.1 按功能分包目录结构

适合中小项目：

```text
app/src/main/java/com/example/app/
  user/
    data/
      UserDto.kt
      UserApi.kt
      UserRepository.kt
    ui/
      UserScreen.kt
      UserViewModel.kt
      UserUiState.kt
  core/
    network/
      RetrofitFactory.kt
```

### 8.2 按层分包目录结构

适合团队有明确层级规范的项目：

```text
app/src/main/java/com/example/app/
  data/
    user/
      UserDto.kt
      UserApi.kt
      UserRepository.kt
  ui/
    user/
      UserScreen.kt
      UserViewModel.kt
      UserUiState.kt
  core/
    network/
      RetrofitFactory.kt
```

建议：Android 业务项目通常优先按功能分包。因为改一个业务时，大多数相关文件在同一目录下。

### 8.3 UiState

```kotlin
data class UserUiState(
    val loading: Boolean = false,
    val users: List<User> = emptyList(),
    val errorMessage: String? = null
)
```

说明：

- `loading` 表示是否加载中。
- `users` 是页面要展示的数据。
- `errorMessage` 保存错误文案。
- UI 只根据 `UserUiState` 渲染，不直接关心网络细节。

### 8.4 Repository

```kotlin
class UserRepository {
    suspend fun getUsers(): List<User> {
        // 真实项目这里可以调用 Retrofit API 或 Room DAO
        delay(500)

        return listOf(
            User(1, "Tom", 18),
            User(2, "Jerry", 20)
        )
    }
}
```

### 8.5 ViewModel

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            // 先进入 loading 状态
            _uiState.value = _uiState.value.copy(
                loading = true,
                errorMessage = null
            )

            runCatching {
                repository.getUsers()
            }.onSuccess { users ->
                _uiState.value = UserUiState(
                    loading = false,
                    users = users
                )
            }.onFailure { error ->
                _uiState.value = UserUiState(
                    loading = false,
                    errorMessage = error.message ?: "Load failed"
                )
            }
        }
    }
}
```

### 8.6 Compose View

```kotlin
@Composable
fun UserRoute(
    viewModel: UserViewModel = viewModel()
) {
    val state by viewModel.uiState.collectAsState()

    LaunchedEffect(Unit) {
        viewModel.loadUsers()
    }

    UserScreen(state = state)
}
```

```kotlin
@Composable
fun UserScreen(state: UserUiState) {
    when {
        state.loading -> {
            CircularProgressIndicator()
        }

        state.errorMessage != null -> {
            Text(text = state.errorMessage)
        }

        else -> {
            LazyColumn {
                items(state.users) { user ->
                    Text(text = "${user.name}, ${user.age}")
                }
            }
        }
    }
}
```

### 8.7 优缺点

优点：

- 适合 Android 生命周期。
- ViewModel 自动适配配置变更。
- UI 状态集中，页面更可预测。
- 易于和 Compose、Flow、Repository 配合。

缺点：

- 初学者容易把所有业务都塞进 ViewModel。
- 没有统一事件模型时，复杂页面可能变乱。
- Repository 抽象过度会产生样板代码。

适合：大多数现代 Android 项目，尤其 Kotlin + Compose 项目。

## 9. MVI / 单向数据流

MVI 常见概念：

| 概念 | 说明 |
|------|------|
| Model / State | 页面完整状态 |
| View | 根据 State 渲染 UI |
| Intent / Action | 用户意图或页面事件 |
| Reducer | 根据旧状态和事件生成新状态 |
| Effect | 一次性副作用，例如 Toast、导航 |

数据流：

```text
UserAction
    |
    v
ViewModel
    |
    v
StateFlow<UiState>
    |
    v
UI render
```

### 9.1 目录结构

```text
app/src/main/java/com/example/app/user/
  data/
    UserRepository.kt
  mvi/
    UserAction.kt
    UserEffect.kt
    UserUiState.kt
    UserViewModel.kt
    UserScreen.kt
```

### 9.2 State

```kotlin
data class UserUiState(
    val loading: Boolean = false,
    val users: List<User> = emptyList(),
    val errorMessage: String? = null
)
```

### 9.3 Action

```kotlin
sealed interface UserAction {
    // 页面首次进入
    data object Load : UserAction

    // 用户点击重试
    data object Retry : UserAction

    // 用户点击某个用户
    data class UserClicked(val userId: Long) : UserAction
}
```

### 9.4 Effect

```kotlin
sealed interface UserEffect {
    // 一次性导航事件，不适合放在持久 UiState 中
    data class NavigateToDetail(val userId: Long) : UserEffect

    // 一次性提示
    data class ShowToast(val message: String) : UserEffect
}
```

### 9.5 ViewModel

```kotlin
class UserViewModel(
    private val repository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    private val _effect = MutableSharedFlow<UserEffect>()
    val effect: SharedFlow<UserEffect> = _effect.asSharedFlow()

    fun onAction(action: UserAction) {
        when (action) {
            UserAction.Load,
            UserAction.Retry -> loadUsers()

            is UserAction.UserClicked -> {
                viewModelScope.launch {
                    _effect.emit(UserEffect.NavigateToDetail(action.userId))
                }
            }
        }
    }

    private fun loadUsers() {
        viewModelScope.launch {
            reduce {
                copy(
                    loading = true,
                    errorMessage = null
                )
            }

            runCatching {
                repository.getUsers()
            }.onSuccess { users ->
                reduce {
                    copy(
                        loading = false,
                        users = users
                    )
                }
            }.onFailure { error ->
                reduce {
                    copy(
                        loading = false,
                        errorMessage = error.message ?: "Load failed"
                    )
                }

                _effect.emit(UserEffect.ShowToast("Load failed"))
            }
        }
    }

    private fun reduce(block: UserUiState.() -> UserUiState) {
        _uiState.value = _uiState.value.block()
    }
}
```

### 9.6 Compose View

```kotlin
@Composable
fun UserRoute(
    viewModel: UserViewModel,
    onNavigateToDetail: (Long) -> Unit
) {
    val state by viewModel.uiState.collectAsState()
    val context = LocalContext.current

    LaunchedEffect(Unit) {
        viewModel.onAction(UserAction.Load)
    }

    LaunchedEffect(Unit) {
        viewModel.effect.collect { effect ->
            when (effect) {
                is UserEffect.NavigateToDetail -> {
                    onNavigateToDetail(effect.userId)
                }

                is UserEffect.ShowToast -> {
                    Toast.makeText(context, effect.message, Toast.LENGTH_SHORT).show()
                }
            }
        }
    }

    UserScreen(
        state = state,
        onRetry = { viewModel.onAction(UserAction.Retry) },
        onUserClick = { userId ->
            viewModel.onAction(UserAction.UserClicked(userId))
        }
    )
}
```

```kotlin
@Composable
fun UserScreen(
    state: UserUiState,
    onRetry: () -> Unit,
    onUserClick: (Long) -> Unit
) {
    when {
        state.loading -> CircularProgressIndicator()

        state.errorMessage != null -> {
            Column {
                Text(state.errorMessage)
                Button(onClick = onRetry) {
                    Text("Retry")
                }
            }
        }

        else -> {
            LazyColumn {
                items(state.users) { user ->
                    Text(
                        text = user.name,
                        modifier = Modifier.clickable {
                            onUserClick(user.id)
                        }
                    )
                }
            }
        }
    }
}
```

### 9.7 优缺点

优点：

- 状态变化路径清晰。
- 适合复杂页面和 Compose。
- 事件和一次性副作用分开，减少重复导航和 Toast 问题。
- 易于记录和回放用户行为。

缺点：

- 样板代码比 MVVM 多。
- 简单页面会显得重。
- 团队需要统一命名和事件拆分规则。

适合：状态复杂、交互多、需要强一致 UI 状态的项目。

## 10. Clean Architecture

Clean Architecture 强调依赖规则：

```text
Presentation
      |
      v
Domain
      ^
      |
Data
```

更准确地说：

- Presentation 可以依赖 Domain。
- Data 可以依赖 Domain 中定义的接口和实体。
- Domain 不依赖 Android、不依赖 Retrofit、不依赖 Room。

### 10.1 目录结构，单模块版

```text
app/src/main/java/com/example/app/
  presentation/
    user/
      UserScreen.kt
      UserViewModel.kt
      UserUiState.kt
  domain/
    model/
      User.kt
    repository/
      UserRepository.kt
    usecase/
      GetUsersUseCase.kt
  data/
    remote/
      UserApi.kt
      UserDto.kt
    local/
      UserDao.kt
      UserEntity.kt
    repository/
      UserRepositoryImpl.kt
```

### 10.2 Domain Model

```kotlin
data class User(
    val id: Long,
    val name: String,
    val age: Int
)
```

### 10.3 Repository 接口，放在 Domain

```kotlin
interface UserRepository {
    suspend fun getUsers(): List<User>
}
```

说明：

- Domain 只关心“我要用户列表”。
- Domain 不关心数据来自网络、数据库还是缓存。

### 10.4 UseCase

```kotlin
class GetUsersUseCase(
    private val repository: UserRepository
) {
    suspend operator fun invoke(): List<User> {
        // 这里可以放业务规则，例如过滤无效用户
        return repository.getUsers()
            .filter { user -> user.name.isNotBlank() }
    }
}
```

说明：

- `operator fun invoke()` 让 UseCase 可以像函数一样调用。
- UseCase 不应该知道 Android UI。
- UseCase 适合放可复用业务规则，不要把每个 Repository 方法都机械包一层。

### 10.5 Data DTO

```kotlin
data class UserDto(
    val id: Long,
    val name: String?,
    val age: Int?
)
```

DTO 转 Domain：

```kotlin
fun UserDto.toDomain(): User {
    return User(
        id = id,
        name = name.orEmpty(),
        age = age ?: 0
    )
}
```

说明：

- 网络字段可能为空，所以 DTO 更贴近接口真实情况。
- Domain Model 应尽量保持业务安全，例如 `name` 不为空。

### 10.6 Data Repository 实现

```kotlin
class UserRepositoryImpl(
    private val api: UserApi,
    private val dao: UserDao
) : UserRepository {

    override suspend fun getUsers(): List<User> {
        return try {
            // 先请求网络
            val remoteUsers = api.getUsers()

            // 保存到本地数据库，便于离线使用
            dao.saveUsers(remoteUsers.map { it.toEntity() })

            remoteUsers.map { it.toDomain() }
        } catch (e: Exception) {
            // 网络失败时读取本地缓存
            dao.getUsers().map { it.toDomain() }
        }
    }
}
```

### 10.7 Presentation ViewModel

```kotlin
class UserViewModel(
    private val getUsersUseCase: GetUsersUseCase
) : ViewModel() {

    private val _uiState = MutableStateFlow(UserUiState())
    val uiState: StateFlow<UserUiState> = _uiState.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            _uiState.value = _uiState.value.copy(loading = true)

            runCatching {
                getUsersUseCase()
            }.onSuccess { users ->
                _uiState.value = UserUiState(users = users)
            }.onFailure { error ->
                _uiState.value = UserUiState(
                    errorMessage = error.message ?: "Load failed"
                )
            }
        }
    }
}
```

### 10.8 优缺点

优点：

- 业务规则独立于 Android 框架。
- 测试 Domain 层很容易。
- 数据来源可替换。
- 适合复杂业务和多人协作。

缺点：

- 目录和类更多。
- 简单 CRUD 页面可能过度设计。
- 如果团队只照搬模板，会产生很多无意义 UseCase。

适合：中大型项目、复杂业务、强测试需求。

## 11. 多模块 Clean Architecture

当项目变大，可以把层拆成 Gradle 模块。

### 11.1 目录结构

```text
root/
  app/
  core/
    common/
    network/
    database/
    ui/
  feature/
    user/
      presentation/
      domain/
      data/
    order/
      presentation/
      domain/
      data/
```

或者更严格：

```text
root/
  app/
  core/
    common/
    network/
    database/
    designsystem/
  feature-user/
    src/main/java/.../user/
      presentation/
      domain/
      data/
  feature-order/
    src/main/java/.../order/
      presentation/
      domain/
      data/
```

### 11.2 模块依赖

```text
app
 |
 v
feature:user
 |
 |----> core:ui
 |----> core:common
 |----> core:network
 |----> core:database
```

更严格的 Clean 多模块：

```text
feature:user:presentation -> feature:user:domain
feature:user:data         -> feature:user:domain
feature:user:data         -> core:network
feature:user:data         -> core:database
```

### 11.3 build.gradle.kts 示例

`feature/user/build.gradle.kts`：

```kotlin
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
}

android {
    namespace = "com.example.feature.user"
    compileSdk = 35
}

dependencies {
    implementation(project(":core:common"))
    implementation(project(":core:network"))
    implementation(project(":core:database"))
    implementation(project(":core:ui"))
}
```

### 11.4 优缺点

优点：

- 业务边界更清晰。
- 可以减少无关模块编译。
- 团队协作冲突更少。
- 方便做动态特性模块。

缺点：

- Gradle 配置更复杂。
- 模块边界设计不好会导致依赖混乱。
- 初期项目拆太细会降低效率。

适合：业务线多、团队多人、构建变慢、代码规模较大的项目。

## 12. 按层分包 vs 按功能分包

### 12.1 按层分包

```text
com.example.app/
  ui/
    user/
    order/
  data/
    user/
    order/
  domain/
    user/
    order/
```

优点：

- 层级很清楚。
- 适合架构教学。

缺点：

- 改一个用户功能要在多个顶层目录跳来跳去。
- 业务边界不如按功能直观。

### 12.2 按功能分包

```text
com.example.app/
  user/
    ui/
    domain/
    data/
  order/
    ui/
    domain/
    data/
  core/
    network/
    database/
```

优点：

- 业务聚合度高。
- 改一个功能时文件集中。
- 更容易迁移到 feature module。

缺点：

- 如果没有规范，每个 feature 内部结构可能不一致。

建议：业务项目优先按功能分包，公共能力放 `core/`。

## 13. Hilt 依赖注入下的目录和代码

架构分层后，依赖创建会变多。Hilt 可以统一管理对象创建。

### 13.1 目录结构

```text
app/src/main/java/com/example/app/
  di/
    NetworkModule.kt
    RepositoryModule.kt
  user/
    data/
      UserRepositoryImpl.kt
    domain/
      UserRepository.kt
      GetUsersUseCase.kt
    ui/
      UserViewModel.kt
```

### 13.2 RepositoryModule

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl
    ): UserRepository
}
```

说明：

- `@Module` 表示这是 Hilt 依赖提供模块。
- `@Binds` 表示把接口绑定到具体实现。
- 外部依赖 `UserRepository` 时，Hilt 会注入 `UserRepositoryImpl`。

### 13.3 ViewModel 注入

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val getUsersUseCase: GetUsersUseCase
) : ViewModel() {
    // ViewModel 代码同前面示例
}
```

### 13.4 Activity

```kotlin
@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        setContent {
            val viewModel: UserViewModel = hiltViewModel()
            UserRoute(viewModel = viewModel)
        }
    }
}
```

## 14. 架构中的数据模型划分

中大型项目不要所有层共用一个 Model。

```text
Network DTO -> Entity -> Domain Model -> Ui Model
```

| 类型 | 示例 | 作用 |
|------|------|------|
| DTO | `UserDto` | 接口返回结构 |
| Entity | `UserEntity` | 数据库存储结构 |
| Domain Model | `User` | 业务层模型 |
| Ui Model | `UserUiModel` | UI 展示模型 |

示例：

```kotlin
data class UserDto(
    val id: Long,
    val first_name: String?,
    val last_name: String?
)

data class UserEntity(
    val id: Long,
    val fullName: String
)

data class User(
    val id: Long,
    val name: String
)

data class UserUiModel(
    val title: String,
    val subtitle: String
)
```

转换：

```kotlin
fun UserDto.toEntity(): UserEntity {
    return UserEntity(
        id = id,
        fullName = listOfNotNull(first_name, last_name)
            .joinToString(" ")
    )
}

fun UserEntity.toDomain(): User {
    return User(
        id = id,
        name = fullName
    )
}

fun User.toUiModel(): UserUiModel {
    return UserUiModel(
        title = name,
        subtitle = "ID: $id"
    )
}
```

小项目可以减少模型层数，但要知道每种模型的边界。

## 15. 测试角度看架构

### 15.1 Repository 测试

```kotlin
class FakeUserRepository : UserRepository {
    override suspend fun getUsers(): List<User> {
        return listOf(User(1, "Tom", 18))
    }
}
```

### 15.2 UseCase 测试

```kotlin
class GetUsersUseCaseTest {
    @Test
    fun invoke_filtersBlankNameUsers() = runTest {
        val repository = object : UserRepository {
            override suspend fun getUsers(): List<User> {
                return listOf(
                    User(1, "Tom", 18),
                    User(2, "", 20)
                )
            }
        }

        val useCase = GetUsersUseCase(repository)

        val result = useCase()

        assertEquals(1, result.size)
        assertEquals("Tom", result.first().name)
    }
}
```

### 15.3 ViewModel 测试思路

```kotlin
class UserViewModelTest {
    @Test
    fun loadUsers_success_updatesState() = runTest {
        val repository = FakeUserRepository()
        val viewModel = UserViewModel(repository)

        viewModel.loadUsers()

        // 实际项目需要配合测试 dispatcher 和 Turbine 测 StateFlow
        val state = viewModel.uiState.value
        assertEquals(false, state.loading)
        assertEquals("Tom", state.users.first().name)
    }
}
```

架构好的代码，越靠近业务规则越容易测试。

## 16. 常见反模式

| 反模式 | 问题 |
|--------|------|
| Fat Activity / Fat Fragment | 生命周期、UI、网络、业务都混在一起 |
| God ViewModel | ViewModel 包含所有业务规则和数据细节 |
| Repository 什么都管 | Repository 变成新的 God Class |
| 每个方法一个 UseCase | 样板代码爆炸，没有真实业务价值 |
| Domain 依赖 Android Context | 业务层被 Android 框架污染 |
| UI 直接使用 DTO | 接口变化直接影响 UI |
| 模块拆太细 | 构建和依赖管理成本高于收益 |
| 过早引入复杂架构 | 小项目开发效率下降 |

## 17. 架构演进建议

### 17.1 小项目

```text
feature/
  ui/
  data/
```

建议：

- MVVM。
- Repository 简单封装数据。
- 暂时不强制 UseCase。

### 17.2 中型项目

```text
feature/
  ui/
  domain/
  data/
core/
  network/
  database/
  ui/
```

建议：

- MVVM 或 MVI。
- 复杂业务引入 UseCase。
- 使用 Hilt。
- DTO、Entity、Domain Model 适当分离。

### 17.3 大型项目

```text
app/
core:*
feature:*
```

建议：

- 多模块。
- 功能模块边界明确。
- 统一架构模板。
- CI 中执行 lint、test、build。
- 依赖方向用 Gradle 约束。

## 18. 新手推荐落地模板

如果你是初学者，推荐从这个结构开始：

```text
app/src/main/java/com/example/app/
  core/
    network/
      ApiClient.kt
    database/
      AppDatabase.kt
  user/
    data/
      UserRepository.kt
      UserApi.kt
      UserDto.kt
    ui/
      UserScreen.kt
      UserViewModel.kt
      UserUiState.kt
```

先不要急着引入 Clean Architecture。等你发现这些信号再升级：

- 同一个业务规则被多个 ViewModel 复制。
- Repository 里开始出现大量业务判断。
- 数据来源变多，网络、本地、缓存合并逻辑复杂。
- 单元测试很难写。

升级后：

```text
user/
  data/
  domain/
  ui/
```

## 19. 架构对比总结

| 架构 | 目录复杂度 | 测试友好 | 适合 Compose | 适合老项目 | 推荐程度 |
|------|------------|----------|--------------|------------|----------|
| 无架构 | 低 | 差 | 一般 | 一般 | Demo 可用 |
| MVC | 低 | 较差 | 一般 | 常见 | 了解 |
| MVP | 中 | 中 | 一般 | 适合 | 维护老项目 |
| MVVM | 中 | 好 | 好 | 可用 | 推荐 |
| MVI | 中高 | 好 | 很好 | 一般 | 状态复杂时推荐 |
| Clean | 高 | 很好 | 好 | 可用 | 中大型项目推荐 |
| 多模块 | 高 | 取决于设计 | 好 | 改造成本高 | 大型项目推荐 |

## 20. 参考资料

- Android App architecture：https://developer.android.com/topic/architecture
- Android UI layer：https://developer.android.com/topic/architecture/ui-layer
- Android Data layer：https://developer.android.com/topic/architecture/data-layer
- Android Domain layer：https://developer.android.com/topic/architecture/domain-layer
- Android Modularization：https://developer.android.com/topic/modularization
- Android Dependency injection：https://developer.android.com/training/dependency-injection

## 21. 总结

Android 架构学习的重点不是背名词，而是理解职责拆分和依赖方向：

```text
UI 负责展示
ViewModel/Presenter 负责页面逻辑
UseCase 负责业务规则
Repository 负责数据协调
DataSource/Api/Dao 负责具体数据来源
```

新手最推荐的路径：

```text
先学 MVVM
  |
  v
用 UiState 管页面状态
  |
  v
用 Repository 隔离数据来源
  |
  v
复杂业务再引入 UseCase
  |
  v
页面状态复杂后学习 MVI
  |
  v
项目变大后学习模块化
```
