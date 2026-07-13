# Android 进阶知识：渲染、运行时、性能、安全与音视频

## 1. 与 iOS 知识点的对应关系

| iOS 主题 | Android 对应主题 |
|----------|------------------|
| UIView / CALayer 渲染 | View 绘制流程、RenderThread、Surface、Canvas、Compose 渲染 |
| RunLoop | Looper、MessageQueue、Handler、Choreographer |
| Runtime | ART、ClassLoader、JNI、反射、注解处理 |
| 性能优化 | 启动、内存、卡顿、包体积、I/O、网络、电量 |
| 安全机制 | Keystore、Sandbox、权限、签名、混淆、Root/Hook 风险 |
| 音视频开发 | MediaPlayer、ExoPlayer、CameraX、MediaCodec、AudioRecord、OpenGL/Surface |

## 2. Android UI 渲染机制

传统 View 的核心流程：

```text
Activity.setContentView
        |
        v
ViewRootImpl
        |
        v
measure -> layout -> draw
        |
        v
DisplayList / RenderNode
        |
        v
RenderThread + GPU
        |
        v
SurfaceFlinger 合成上屏
```

三个阶段：

| 阶段 | 作用 |
|------|------|
| `measure` | 计算 View 宽高 |
| `layout` | 确定 View 在父容器中的位置 |
| `draw` | 绘制背景、内容、子 View、前景 |

自定义 View 示例：

```kotlin
class ProgressBarView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs) {

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.BLUE
        style = Paint.Style.FILL
    }

    var progress: Float = 0.5f
        set(value) {
            // 限制范围，避免传入非法值
            field = value.coerceIn(0f, 1f)

            // 触发重新绘制，只会走 draw，不一定重新 measure/layout
            invalidate()
        }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        val width = width * progress

        // 在当前 View 的 Canvas 上绘制进度条
        canvas.drawRect(
            0f,
            0f,
            width,
            height.toFloat(),
            paint
        )
    }
}
```

性能要点：

- `onDraw()` 不要创建大量对象。
- 只需要重绘时用 `invalidate()`。
- 尺寸变化时再用 `requestLayout()`。
- 列表中避免复杂嵌套布局。
- Compose 中避免不必要的状态上提和重复重组。

## 3. Looper / Handler / Choreographer

Android 主线程事件模型：

```text
main thread
  |
  v
Looper.loop()
  |
  v
MessageQueue
  |
  v
Handler.dispatchMessage()
```

Handler 示例：

```kotlin
class MainActivity : AppCompatActivity() {
    private val mainHandler = Handler(Looper.getMainLooper())

    fun loadData() {
        Thread {
            // 子线程执行耗时任务
            val result = "data loaded"

            mainHandler.post {
                // 回到主线程更新 UI
                findViewById<TextView>(R.id.title).text = result
            }
        }.start()
    }
}
```

Choreographer 和屏幕刷新相关：

```kotlin
Choreographer.getInstance().postFrameCallback { frameTimeNanos ->
    // 每一帧回调，可用于动画或卡顿监控
    Log.d("Frame", "frameTime=$frameTimeNanos")
}
```

卡顿理解：

```text
16.6ms 一帧，60Hz 屏幕
如果主线程 measure/layout/draw/input/业务执行过久
就会错过下一次 VSync，产生掉帧
```

## 4. ART、ClassLoader 与运行时

Android Runtime 关键点：

| 概念 | 说明 |
|------|------|
| ART | Android Runtime，执行应用字节码 |
| DEX | Android 字节码文件格式 |
| ClassLoader | 加载类，插件化和热修复常涉及 |
| JNI | Java/Kotlin 调用 C/C++ |
| 反射 | 运行时访问类、方法、字段 |

ClassLoader 简化理解：

```text
BootClassLoader
      |
      v
PathClassLoader，加载 APK 内 classes.dex
      |
      v
DexClassLoader，可加载外部 dex/jar/apk
```

反射示例：

```kotlin
data class User(val name: String)

fun readNameByReflection(user: User): String {
    val field = user.javaClass.getDeclaredField("name")

    // private 字段需要打开访问权限
    field.isAccessible = true

    return field.get(user) as String
}
```

注意：

- 反射慢且容易被混淆影响。
- Android 高版本对非公开 API 有限制。
- 插件化、热修复要关注签名、安全和兼容性。

## 5. 启动性能优化

启动阶段：

```text
点击图标
  |
  v
Zygote fork 进程
  |
  v
Application.attachBaseContext
  |
  v
Application.onCreate
  |
  v
Activity.onCreate
  |
  v
首帧绘制
```

常见问题：

- `Application.onCreate()` 初始化太多 SDK。
- 主线程读写文件、数据库。
- 首页布局复杂。
- 首屏同步等待网络。
- 冷启动加载大量类。

优化策略：

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()

        // 只初始化启动必需项
        initCrashReporter()

        // 非关键 SDK 延后
        Handler(Looper.getMainLooper()).post {
            initAnalytics()
        }
    }
}
```

建议：

- SDK 分级：启动必需、首屏后、用户触发。
- 首页先展示骨架屏或缓存。
- 避免主线程 I/O。
- 使用 Baseline Profile 优化启动和关键路径。

## 6. 内存与泄漏

常见内存问题：

| 问题 | 示例 |
|------|------|
| Activity 泄漏 | 单例持有 Activity |
| Bitmap 过大 | 原图直接加载到内存 |
| 集合缓存无限增长 | Map/List 不清理 |
| Handler 延迟任务 | Runnable 持有页面 |
| 监听未注销 | BroadcastReceiver、Callback |

泄漏示例：

```kotlin
object UserManager {
    // 错误：单例持有 Activity 会导致页面无法释放
    var context: Context? = null
}
```

修正：

```kotlin
object UserManager {
    private lateinit var appContext: Context

    fun init(context: Context) {
        // 使用 applicationContext，避免持有 Activity
        appContext = context.applicationContext
    }
}
```

图片建议：

- 使用 Coil / Glide。
- 按目标尺寸加载。
- 列表中使用缩略图。
- 大图预览要分页或瓦片加载。

## 7. UI 流畅度优化

列表优化：

```kotlin
class UserAdapter : ListAdapter<User, UserViewHolder>(DIFF) {
    companion object {
        val DIFF = object : DiffUtil.ItemCallback<User>() {
            override fun areItemsTheSame(oldItem: User, newItem: User): Boolean {
                return oldItem.id == newItem.id
            }

            override fun areContentsTheSame(oldItem: User, newItem: User): Boolean {
                return oldItem == newItem
            }
        }
    }
}
```

要点：

- RecyclerView 使用 DiffUtil。
- Compose LazyColumn 给稳定 key。
- 避免在绑定 View 时做耗时计算。
- 图片异步加载。
- 复杂计算提前到 ViewModel 或后台线程。

Compose 示例：

```kotlin
LazyColumn {
    items(
        items = users,
        key = { user -> user.id }
    ) { user ->
        Text(text = user.name)
    }
}
```

## 8. 包体积优化

主要构成：

```text
classes.dex
native .so
resources.arsc
res 图片和布局
assets
第三方库
```

常用策略：

```kotlin
android {
    buildTypes {
        release {
            isMinifyEnabled = true
            isShrinkResources = true
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
}
```

建议：

- 开启 R8 混淆和资源压缩。
- 删除无用资源。
- 图片使用 WebP / AVIF。
- 按 ABI 拆分 native 包。
- 避免引入过重 SDK。
- 用 Android Studio APK Analyzer 分析。

## 9. Android 安全机制

安全基础：

| 机制 | 说明 |
|------|------|
| Sandbox | 每个 App 独立 UID 和私有目录 |
| Permission | 权限声明和运行时授权 |
| App Signing | APK/AAB 签名保证来源和完整性 |
| Keystore | 硬件或系统保护密钥 |
| Network Security Config | 控制明文和证书策略 |
| R8/ProGuard | 混淆压缩，增加逆向成本 |

Keystore 示例：

```kotlin
fun createAesKey(alias: String) {
    val keyGenerator = KeyGenerator.getInstance(
        KeyProperties.KEY_ALGORITHM_AES,
        "AndroidKeyStore"
    )

    val spec = KeyGenParameterSpec.Builder(
        alias,
        KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
    )
        .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
        .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
        .build()

    keyGenerator.init(spec)
    keyGenerator.generateKey()
}
```

网络安全配置：

```xml
<!-- res/xml/network_security_config.xml -->
<network-security-config>
    <base-config cleartextTrafficPermitted="false" />
</network-security-config>
```

Manifest：

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config">
</application>
```

安全建议：

- Token 不要明文落盘。
- 不要在日志里打印隐私数据。
- WebView 禁用不必要能力。
- 导出组件必须检查 `android:exported`。
- 服务端仍要做鉴权，不能只信客户端。

## 10. 音视频开发

常用 API：

| 场景 | 推荐 |
|------|------|
| 普通播放 | ExoPlayer / Media3 |
| 简单系统播放 | MediaPlayer |
| 相机采集 | CameraX |
| 编解码 | MediaCodec |
| 音频录制 | AudioRecord |
| 视频渲染 | SurfaceView / TextureView / OpenGL |

CameraX 拍照示例：

```kotlin
class CameraActivity : AppCompatActivity() {
    private lateinit var imageCapture: ImageCapture

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(this)

        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()

            val preview = Preview.Builder().build().also {
                it.setSurfaceProvider(previewView.surfaceProvider)
            }

            imageCapture = ImageCapture.Builder().build()

            cameraProvider.bindToLifecycle(
                this,
                CameraSelector.DEFAULT_BACK_CAMERA,
                preview,
                imageCapture
            )
        }, ContextCompat.getMainExecutor(this))
    }
}
```

播放器状态要点：

- 生命周期中暂停/释放播放器。
- 后台播放使用前台服务和通知。
- 大视频首帧优化要关注缓存和预加载。
- 直播关注延迟、弱网、丢帧策略。

## 11. 常用性能工具

| 工具 | 用途 |
|------|------|
| Android Studio Profiler | CPU、内存、网络、电量 |
| Layout Inspector | 查看 View/Compose 层级 |
| APK Analyzer | 包体积分析 |
| Perfetto / Systrace | 系统级性能追踪 |
| LeakCanary | 内存泄漏检测 |
| Macrobenchmark | 启动和滚动等场景基准测试 |
| Baseline Profile | 优化启动和关键路径执行 |

## 12. 可实践示例：自定义 View 的测量、布局与绘制

前面的 `ProgressBarView` 只演示了 `onDraw()`。真实自定义 View 至少要考虑：

- `onMeasure()`：没有明确尺寸时给出默认宽高。
- `onDraw()`：只负责绘制，不做业务请求和重计算。
- 自定义属性：让 XML 或代码能配置颜色、高度、进度。
- 状态更新：进度变化时只重绘，不重新布局。

完整示例：

```kotlin
class SegmentedProgressView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : View(context, attrs) {

    private val backgroundPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.LTGRAY
        style = Paint.Style.FILL
    }

    private val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        color = Color.rgb(33, 150, 243)
        style = Paint.Style.FILL
    }

    private val cornerRadius = 12f

    var progress: Float = 0f
        set(value) {
            val newValue = value.coerceIn(0f, 1f)

            if (field == newValue) {
                return
            }

            field = newValue

            // 进度只影响绘制宽度，不影响 View 尺寸，所以使用 invalidate
            invalidate()
        }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val defaultWidth = dp(240)
        val defaultHeight = dp(12)

        val measuredWidth = resolveSize(defaultWidth, widthMeasureSpec)
        val measuredHeight = resolveSize(defaultHeight, heightMeasureSpec)

        setMeasuredDimension(measuredWidth, measuredHeight)
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        val fullRect = RectF(
            0f,
            0f,
            width.toFloat(),
            height.toFloat()
        )

        // 绘制背景
        canvas.drawRoundRect(
            fullRect,
            cornerRadius,
            cornerRadius,
            backgroundPaint
        )

        val progressRect = RectF(
            0f,
            0f,
            width * progress,
            height.toFloat()
        )

        // 绘制进度
        canvas.drawRoundRect(
            progressRect,
            cornerRadius,
            cornerRadius,
            progressPaint
        )
    }

    private fun dp(value: Int): Int {
        return (value * resources.displayMetrics.density).toInt()
    }
}
```

在 Activity 中使用：

```kotlin
class ProgressActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val progressView = SegmentedProgressView(this).apply {
            progress = 0.3f
        }

        setContentView(progressView)

        lifecycleScope.launch {
            repeat(10) { index ->
                delay(300)

                // 每次更新都会触发 invalidate，重新绘制进度
                progressView.progress = (index + 1) / 10f
            }
        }
    }
}
```

排查清单：

- View 不显示：检查 `onMeasure()` 是否给了非 0 尺寸。
- 改了数据不刷新：检查是否调用 `invalidate()` 或 `requestLayout()`。
- 滚动卡顿：检查 `onDraw()` 是否频繁创建对象。
- 圆角异常：检查进度宽度小于圆角时是否符合视觉预期。

## 13. 可实践示例：主线程卡顿监控

Android 卡顿的本质通常是主线程消息执行时间过长。一个简单卡顿监控可以通过 `Looper.setMessageLogging()` 观察每个消息的开始和结束。

```kotlin
object MainThreadBlockMonitor {
    private const val TAG = "BlockMonitor"
    private const val THRESHOLD_MS = 100L

    private var startTime = 0L
    private var startLog = ""

    fun install() {
        Looper.getMainLooper().setMessageLogging { log ->
            if (log.startsWith(">>>>>")) {
                // 一条主线程消息开始执行
                startTime = SystemClock.uptimeMillis()
                startLog = log
            } else if (log.startsWith("<<<<<")) {
                // 一条主线程消息执行结束
                val cost = SystemClock.uptimeMillis() - startTime

                if (cost > THRESHOLD_MS) {
                    Log.w(
                        TAG,
                        "main thread message cost=${cost}ms, start=$startLog"
                    )
                }
            }
        }
    }
}
```

在 `Application` 中启用：

```kotlin
class App : Application() {
    override fun onCreate() {
        super.onCreate()

        if (BuildConfig.DEBUG) {
            MainThreadBlockMonitor.install()
        }
    }
}
```

说明：

- 这个方案适合学习和 Debug，不是完整 APM。
- 真实线上卡顿监控还要采集堆栈、页面、设备信息、采样频率。
- 卡顿阈值要按场景调整，100ms 只是示例。

## 14. 可实践示例：启动耗时埋点

启动优化不能只靠感觉，需要先测量。

Application：

```kotlin
class App : Application() {
    override fun attachBaseContext(base: Context) {
        LaunchTracker.mark("attachBaseContext_start")
        super.attachBaseContext(base)
        LaunchTracker.mark("attachBaseContext_end")
    }

    override fun onCreate() {
        LaunchTracker.mark("app_onCreate_start")
        super.onCreate()

        initRequiredSdk()

        LaunchTracker.mark("app_onCreate_end")
    }

    private fun initRequiredSdk() {
        // 这里只放启动必须的 SDK
    }
}
```

启动追踪器：

```kotlin
object LaunchTracker {
    private val points = mutableListOf<Pair<String, Long>>()

    fun mark(name: String) {
        points += name to SystemClock.uptimeMillis()
    }

    fun dump(tag: String = "Launch") {
        if (points.isEmpty()) return

        val first = points.first().second

        points.forEach { (name, time) ->
            Log.d(tag, "$name costFromStart=${time - first}ms")
        }
    }
}
```

首页首帧：

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        LaunchTracker.mark("activity_onCreate_start")
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        window.decorView.post {
            // decorView 首次 post 通常可粗略代表首帧后时机
            LaunchTracker.mark("first_frame_after")
            LaunchTracker.dump()
        }
    }
}
```

进一步优化：

- 把 SDK 初始化分为 P0/P1/P2。
- P0：崩溃、基础配置。
- P1：首屏后需要。
- P2：用户触发时再初始化。

## 15. 可实践示例：Keystore 加解密

目标：把敏感字符串加密后再存储，密钥由 Android Keystore 管理。

```kotlin
object SecureTextStore {
    private const val KEY_ALIAS = "user_token_key"
    private const val ANDROID_KEYSTORE = "AndroidKeyStore"
    private const val TRANSFORMATION = "AES/GCM/NoPadding"

    fun encrypt(plainText: String): Pair<ByteArray, ByteArray> {
        val secretKey = getOrCreateKey()

        val cipher = Cipher.getInstance(TRANSFORMATION)
        cipher.init(Cipher.ENCRYPT_MODE, secretKey)

        val cipherText = cipher.doFinal(
            plainText.toByteArray(Charsets.UTF_8)
        )

        // GCM 模式需要保存 iv，解密时必须使用同一个 iv
        return cipher.iv to cipherText
    }

    fun decrypt(iv: ByteArray, cipherText: ByteArray): String {
        val secretKey = getOrCreateKey()

        val cipher = Cipher.getInstance(TRANSFORMATION)
        val spec = GCMParameterSpec(128, iv)
        cipher.init(Cipher.DECRYPT_MODE, secretKey, spec)

        val plainBytes = cipher.doFinal(cipherText)
        return plainBytes.toString(Charsets.UTF_8)
    }

    private fun getOrCreateKey(): SecretKey {
        val keyStore = KeyStore.getInstance(ANDROID_KEYSTORE).apply {
            load(null)
        }

        val existingKey = keyStore.getKey(KEY_ALIAS, null) as? SecretKey
        if (existingKey != null) {
            return existingKey
        }

        val keyGenerator = KeyGenerator.getInstance(
            KeyProperties.KEY_ALGORITHM_AES,
            ANDROID_KEYSTORE
        )

        val spec = KeyGenParameterSpec.Builder(
            KEY_ALIAS,
            KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT
        )
            .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
            .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
            .build()

        keyGenerator.init(spec)
        return keyGenerator.generateKey()
    }
}
```

使用：

```kotlin
val (iv, cipherText) = SecureTextStore.encrypt("token_123")
val plainText = SecureTextStore.decrypt(iv, cipherText)
```

注意：

- `iv` 不是密钥，但必须和密文一起保存。
- 不要把密钥导出到普通文件。
- 强安全场景可要求用户认证后才能使用密钥。

## 16. 可实践示例：CameraX 拍照完整流程

权限：

```xml
<uses-permission android:name="android.permission.CAMERA" />
```

Activity：

```kotlin
class CameraActivity : AppCompatActivity() {
    private lateinit var previewView: PreviewView
    private var imageCapture: ImageCapture? = null

    private val requestCamera = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { granted ->
        if (granted) {
            startCamera()
        } else {
            Toast.makeText(this, "Camera permission denied", Toast.LENGTH_SHORT).show()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        previewView = PreviewView(this)
        setContentView(previewView)

        requestCamera.launch(Manifest.permission.CAMERA)
    }

    private fun startCamera() {
        val providerFuture = ProcessCameraProvider.getInstance(this)

        providerFuture.addListener({
            val cameraProvider = providerFuture.get()

            val preview = Preview.Builder().build().apply {
                setSurfaceProvider(previewView.surfaceProvider)
            }

            imageCapture = ImageCapture.Builder()
                .setCaptureMode(ImageCapture.CAPTURE_MODE_MINIMIZE_LATENCY)
                .build()

            cameraProvider.unbindAll()
            cameraProvider.bindToLifecycle(
                this,
                CameraSelector.DEFAULT_BACK_CAMERA,
                preview,
                imageCapture
            )
        }, ContextCompat.getMainExecutor(this))
    }

    private fun takePhoto(file: File) {
        val capture = imageCapture ?: return

        val output = ImageCapture.OutputFileOptions.Builder(file).build()

        capture.takePicture(
            output,
            ContextCompat.getMainExecutor(this),
            object : ImageCapture.OnImageSavedCallback {
                override fun onImageSaved(result: ImageCapture.OutputFileResults) {
                    Toast.makeText(this@CameraActivity, "Saved", Toast.LENGTH_SHORT).show()
                }

                override fun onError(exception: ImageCaptureException) {
                    Log.e("Camera", "take photo failed", exception)
                }
            }
        )
    }
}
```

生命周期要点：

- `bindToLifecycle()` 会跟随 Activity 生命周期自动管理相机。
- 页面退出前不需要手动释放大多数 CameraX 资源。
- 真机必须验证权限、旋转、前后摄、弱光和不同厂商设备。

## 17. 可实践示例：Media3 播放器生命周期

```kotlin
class PlayerActivity : AppCompatActivity() {
    private lateinit var playerView: PlayerView
    private var player: ExoPlayer? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        playerView = PlayerView(this)
        setContentView(playerView)
    }

    override fun onStart() {
        super.onStart()
        initPlayer()
    }

    override fun onStop() {
        super.onStop()
        releasePlayer()
    }

    private fun initPlayer() {
        if (player != null) return

        val mediaItem = MediaItem.fromUri(
            "https://example.com/video.mp4"
        )

        player = ExoPlayer.Builder(this)
            .build()
            .also { exoPlayer ->
                playerView.player = exoPlayer
                exoPlayer.setMediaItem(mediaItem)
                exoPlayer.prepare()
                exoPlayer.playWhenReady = true
            }
    }

    private fun releasePlayer() {
        playerView.player = null
        player?.release()
        player = null
    }
}
```

常见问题：

- 黑屏：检查 URL、网络权限、PlayerView 尺寸。
- 退出后仍有声音：检查 `releasePlayer()` 是否执行。
- 首帧慢：预加载、缓存、服务端首包耗时都要排查。

## 18. 总结

Android 进阶知识可以按这条线理解：

```text
View/Compose 渲染
  |
  v
Looper/Handler/Choreographer 调度
  |
  v
ART/ClassLoader/JNI 运行时
  |
  v
启动、内存、流畅度、包体积优化
  |
  v
Keystore、权限、签名、网络安全
  |
  v
CameraX、Media3、MediaCodec 音视频能力
```

和 iOS 对照学习时，重点不是记 API 名称，而是建立等价心智模型：页面如何绘制、事件如何调度、运行时如何加载代码、性能瓶颈如何定位、安全边界在哪里。

## 19. 参考资料

- Android Performance：https://developer.android.com/topic/performance
- Android Security：https://developer.android.com/privacy-and-security/security-best-practices
- Android Media3：https://developer.android.com/media
- Android CameraX：https://developer.android.com/media/camera/camerax
- Android Runtime：https://source.android.com/docs/core/runtime
