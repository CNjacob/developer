# Flutter 开发、架构与原生集成

## 1. Flutter 技术架构

```text
Application：业务 Widget、状态与路由
Framework：Widgets / Rendering / Animation / Gestures
Engine：Dart Runtime、Skia/Impeller、Text、Platform Channel
Embedder：iOS / Android 生命周期、窗口、线程和输入
Platform：UIKit / Android Framework
```

Flutter 通常自己布局和绘制界面，不把每个 Widget 映射为原生 UIView/View。系统输入、窗口、文本、无障碍和平台能力由 Embedder 与 Engine 协作。

---

## 2. Widget、Element、RenderObject

| 对象 | 职责 | 特点 |
| --- | --- | --- |
| Widget | UI 的不可变配置 | 轻量，可频繁创建 |
| Element | Widget 在树中的实例和上下文 | 管理生命周期与复用 |
| RenderObject | 测量、布局、绘制、命中测试 | 成本较高 |

```text
Widget Tree       配置
    ↓ inflate/update
Element Tree      生命周期和身份
    ↓ create/update
RenderObject Tree 布局与绘制
```

`setState` 标记对应 Element 为 dirty，下一帧重新执行 `build`。重建 Widget 不等于重建 RenderObject。

Key：

- 无 Key：按类型和位置匹配
- `ValueKey`：按稳定业务 ID 匹配
- `ObjectKey`：按对象身份匹配
- `GlobalKey`：跨位置访问 State/Context，成本高，应少用

---

## 3. Build、Layout、Paint 流程

```text
Vsync
  ↓
Animation
  ↓
Build：生成 Widget 配置
  ↓
Layout：约束向下，尺寸向上
  ↓
Paint：生成绘制指令
  ↓
Compositing / Raster
```

Flutter 布局规则：

> Constraints go down. Sizes go up. Parents set positions.

```dart
class ProfileCard extends StatelessWidget {
  const ProfileCard({super.key, required this.user});
  final User user;

  @override
  Widget build(BuildContext context) {
    return Row(
      children: [
        CircleAvatar(backgroundImage: NetworkImage(user.avatarUrl)),
        const SizedBox(width: 12),
        Expanded(
          child: Text(user.name, maxLines: 1, overflow: TextOverflow.ellipsis),
        ),
      ],
    );
  }
}
```

常见问题：

- Row 子节点无限宽：使用 `Expanded`/`Flexible`
- ScrollView 中 Column 高度无限：避免无边界 `Expanded`
- 嵌套大量可滚动组件：明确滚动所有者
- `IntrinsicHeight/Width` 可能触发额外布局，应谨慎

---

## 4. StatefulWidget 生命周期

```text
createState
  ↓
initState
  ↓
didChangeDependencies
  ↓
build
  ↓
didUpdateWidget（配置变化）
  ↓
deactivate
  ↓
dispose
```

```dart
class SearchPageState extends State<SearchPage> {
  final controller = TextEditingController();
  StreamSubscription<Result>? subscription;

  @override
  void initState() {
    super.initState();
    subscription = repository.results.listen(_handleResult);
  }

  @override
  void dispose() {
    subscription?.cancel();
    controller.dispose();
    super.dispose();
  }
}
```

- `initState` 中不能依赖尚未建立的 inherited dependency
- 使用 `context` 订阅依赖通常放在 `didChangeDependencies`
- 异步返回后更新 UI 前检查 `mounted`
- Controller、FocusNode、AnimationController、Timer、Subscription 必须释放

---

## 5. 状态分类

| 状态 | 示例 | 推荐位置 |
| --- | --- | --- |
| 临时 UI 状态 | 展开、选中、动画 | Widget State |
| 页面状态 | 加载、表单、分页 | ViewModel/Notifier |
| 领域状态 | 登录会话、购物车 | Repository/Domain Store |
| 服务端状态 | 列表、缓存、刷新 | Repository/Query 层 |
| 路由状态 | 页面栈、Deep Link | Router |

状态原则：

- 单一可信来源
- 状态不可变
- 事件单向流动
- 不保存可推导数据
- 缩小监听和重建范围

```text
User Action → ViewModel → Repository → New State → View
```

---

## 6. 分层架构

Flutter 官方架构指南推荐 UI 层与 Data 层，并可在复杂业务中增加 Domain 层：

```text
Presentation
  ├── View
  └── ViewModel
        ↓
Domain（可选）
  ├── Entity
  └── UseCase
        ↓
Data
  ├── Repository
  └── Service
        ↓
REST / Database / Platform Plugin
```

职责：

- View：布局、动画、用户事件转发
- ViewModel：页面状态和展示逻辑
- UseCase：跨 Repository 或复杂业务规则
- Repository：领域数据的可信来源、缓存和错误转换
- Service：封装单一外部数据源，无业务状态

---

## 7. Feature 模块示例

```text
lib/
├── app/
│   ├── bootstrap.dart
│   ├── router.dart
│   └── dependencies.dart
├── features/
│   └── order_detail/
│       ├── domain/
│       ├── data/
│       └── presentation/
└── shared/
    ├── network/
    ├── storage/
    └── ui/
```

Domain：

```dart
enum OrderStatus { pending, paid, cancelled }

class Order {
  const Order({required this.id, required this.title, required this.status});
  final String id;
  final String title;
  final OrderStatus status;

  bool get canCancel => status == OrderStatus.pending;
}

abstract interface class OrderRepository {
  Future<Order> find(String id);
  Future<void> cancel(String id);
}
```

ViewModel：

```dart
sealed class OrderState {
  const OrderState();
}
final class OrderLoading extends OrderState {}
final class OrderLoaded extends OrderState {
  const OrderLoaded(this.order, {this.cancelling = false});
  final Order order;
  final bool cancelling;
}
final class OrderFailed extends OrderState {
  const OrderFailed(this.message);
  final String message;
}

class OrderViewModel extends ChangeNotifier {
  OrderViewModel(this.repository, this.orderId);
  final OrderRepository repository;
  final String orderId;

  OrderState state = OrderLoading();

  Future<void> load() async {
    state = OrderLoading();
    notifyListeners();
    try {
      state = OrderLoaded(await repository.find(orderId));
    } catch (error) {
      state = OrderFailed(error.toString());
    }
    notifyListeners();
  }

  Future<void> cancel() async {
    final current = state;
    if (current is! OrderLoaded || !current.order.canCancel) return;
    state = OrderLoaded(current.order, cancelling: true);
    notifyListeners();
    await repository.cancel(current.order.id);
    await load();
  }
}
```

View 通过 Listenable、Provider、Riverpod、Bloc 等方式订阅均可，关键是职责与数据流，不是库名。

---

## 8. 依赖注入

依赖在 Composition Root 创建：

```dart
Future<AppDependencies> createDependencies() async {
  final httpClient = HttpClient(baseUrl: environment.apiBaseUrl);
  final orderService = OrderService(httpClient);
  final orderRepository = RemoteOrderRepository(orderService);
  return AppDependencies(orderRepository: orderRepository);
}
```

不要在 ViewModel 内直接创建 HTTP Client 或读取全局单例。测试时注入 Fake：

```dart
class FakeOrderRepository implements OrderRepository {
  FakeOrderRepository(this.order);
  Order order;

  @override
  Future<Order> find(String id) async => order;

  @override
  Future<void> cancel(String id) async {}
}
```

---

## 9. 路由与 Deep Link

简单页面可使用 Navigator 命令式 API；需要 Web URL、Deep Link、状态恢复或复杂嵌套导航时使用声明式 Router。

```text
External URL
  ↓ Parse + Validate
AppRoute
  ↓ Auth Guard
Navigation State
  ↓
Page Stack
```

路由参数优先传业务 ID，不传大型可变对象。跨 Feature 跳转通过 Router 接口或 App 层编排，Feature 不直接依赖对方内部页面。

---

## 10. 网络、序列化与缓存

```text
ViewModel → Repository → Service → HTTP
                    ↘ Cache
```

Service：

```dart
class OrderService {
  OrderService(this.client);
  final HttpClient client;

  Future<Map<String, Object?>> fetch(String id) {
    return client.getJson('/orders/$id');
  }
}
```

Repository 负责 DTO 映射、缓存策略、重试和领域错误转换。应统一处理：

- Base URL、Header、Token
- 超时与取消
- JSON 校验
- HTTP 错误映射
- Token 刷新并发合并
- 请求日志与 Trace ID
- Cache First / Network First / Stale While Revalidate

---

## 11. Platform Channel

常用 Channel：

| 类型 | 用途 |
| --- | --- |
| `MethodChannel` | 请求—响应方法调用 |
| `EventChannel` | 原生持续向 Dart 发送事件 |
| `BasicMessageChannel` | 双向消息流 |

Dart：

```dart
class BatteryService {
  static const _channel = MethodChannel('com.example/device');

  Future<int> batteryLevel() async {
    final level = await _channel.invokeMethod<int>('batteryLevel');
    if (level == null) throw StateError('Missing battery level');
    return level;
  }
}
```

iOS Swift：

```swift
let channel = FlutterMethodChannel(
    name: "com.example/device",
    binaryMessenger: controller.binaryMessenger
)
channel.setMethodCallHandler { call, result in
    guard call.method == "batteryLevel" else {
        result(FlutterMethodNotImplemented)
        return
    }
    UIDevice.current.isBatteryMonitoringEnabled = true
    result(Int(UIDevice.current.batteryLevel * 100))
}
```

Android Kotlin：

```kotlin
MethodChannel(
    flutterEngine.dartExecutor.binaryMessenger,
    "com.example/device"
).setMethodCallHandler { call, result ->
    if (call.method == "batteryLevel") {
        result.success(readBatteryLevel())
    } else {
        result.notImplemented()
    }
}
```

Channel 名、方法、参数和错误码必须双端一致。大型项目优先使用 Pigeon 生成类型安全契约，避免手写字符串协议漂移。

---

## 12. EventChannel 示例

Dart：

```dart
const connectivityEvents =
    EventChannel('com.example/connectivity/events');

Stream<bool> observeConnectivity() {
  return connectivityEvents
      .receiveBroadcastStream()
      .map((value) => value as bool)
      .distinct();
}
```

原生端必须在取消监听时解除系统监听，避免 Flutter 页面退出后继续持有资源。

```text
onListen → 注册原生监听 → eventSink(...)
onCancel → 解除监听并清空 eventSink
```

---

## 13. Pigeon、Plugin 与 FFI 选择

| 方案 | 适用 |
| --- | --- |
| Platform Channel | 少量平台 API、快速集成 |
| Pigeon | 需要双端强类型生成的 Channel 契约 |
| Flutter Plugin | 可复用、可发布的平台能力 |
| Dart FFI | 调用 C ABI 库和高频原生计算 |
| Platform View | 嵌入地图、相机等原生 UIView/View |

FFI 适合已有 C/C++/Rust 核心库，不适合直接调用任意 Swift/Kotlin 对象。FFI 调用阻塞 Dart Isolate 时仍会卡 UI，应将长计算放到 Worker Isolate 或原生线程。

---

## 14. Add-to-App 混合集成

既有 iOS/Android App 可将 Flutter 作为 Module 集成：

```text
Native App
├── Native Navigation
├── Account / Payment / Existing SDK
└── Flutter Module
    ├── Flutter Features
    └── Host Bridge Plugin
```

创建模块：

```bash
flutter create --template module flutter_module
```

当前官方方式支持：

- Android：Gradle 自动集成或构建 AAR
- iOS：Swift Package、Xcode Build Phase 等集成方式
- 使用 `FlutterEngine` 预热并复用运行环境
- 使用 `flutter attach` 调试混合 App 中的 Dart

宿主应统一管理 Engine 生命周期、入口路由、登录态和 Channel 注册。

---

## 15. iOS 集成模型

```swift
final class FlutterEngineManager {
    static let shared = FlutterEngineManager()

    let engine: FlutterEngine

    private init() {
        engine = FlutterEngine(name: "main_engine")
        engine.run(withEntrypoint: nil)
        GeneratedPluginRegistrant.register(with: engine)
    }
}

func showFlutterPage(route: String) {
    let engine = FlutterEngineManager.shared.engine
    engine.navigationChannel.invokeMethod("pushRoute", arguments: route)
    let controller = FlutterViewController(engine: engine, nibName: nil, bundle: nil)
    navigationController?.pushViewController(controller, animated: true)
}
```

需要设计：

- Engine 是页面级、业务级还是全局复用
- Flutter 页面如何通知原生关闭或跳转
- UINavigationController 与 Flutter Navigator 谁拥有页面栈
- Engine 预热时机与内存成本
- App 生命周期、主题、语言和无障碍同步

一个 Engine 中 Flutter 状态可复用；多个 Engine 状态隔离。不能假设所有 Plugin 都天然支持多 Engine。

---

## 16. Android 集成模型

```kotlin
class App : Application() {
    lateinit var flutterEngine: FlutterEngine

    override fun onCreate() {
        super.onCreate()
        flutterEngine = FlutterEngine(this).apply {
            navigationChannel.setInitialRoute("/home")
            dartExecutor.executeDartEntrypoint(
                DartExecutor.DartEntrypoint.createDefault()
            )
        }
        FlutterEngineCache.getInstance()
            .put("main_engine", flutterEngine)
    }
}
```

```kotlin
startActivity(
    FlutterActivity
        .withCachedEngine("main_engine")
        .build(this)
)
```

需处理 Activity/Fragment 生命周期、返回键、透明背景、状态栏、权限回调和 Plugin 对 Activity 的依赖。

---

## 17. 混合导航协议

推荐定义统一路由消息：

```dart
class NativeRouter {
  static const channel = MethodChannel('com.example/navigation');

  Future<void> openNative(
    String route, {
    Map<String, Object?> arguments = const {},
  }) {
    return channel.invokeMethod('open', {
      'route': route,
      'arguments': arguments,
    });
  }

  Future<void> close([Object? result]) {
    return channel.invokeMethod('close', {'result': result});
  }
}
```

协议必须定义：

- Route 唯一命名和版本
- 参数 Schema
- 页面返回结果
- 登录与权限拦截位置
- 未知路由降级
- Native → Flutter 初始路由传递时机

不要让每个页面单独创建 Channel 和自定义字符串协议。

---

## 18. 模块化与 Package 化

```text
workspace/
├── apps/
│   └── consumer_app/
├── features/
│   ├── account/
│   ├── order/
│   └── payment/
├── foundations/
│   ├── design_system/
│   ├── networking/
│   └── analytics/
└── plugins/
    └── host_bridge/
```

依赖方向：

```text
App → Feature → Foundation
          ↓
       Domain API

Plugin → Platform implementations
```

原则：

- 每个 Package 只有最小公共入口
- Feature 不访问另一个 Feature 的 `src`
- Design System 不依赖业务
- 平台能力集中在 Plugin/Host Bridge
- 避免一个 `common` Package 成为无边界垃圾场
- 通过 CI 检查循环依赖、测试和公共 API

Flutter Add-to-App 在移动端支持多个 Flutter 实例，但官方限制是不支持把多个独立 Flutter Library 打进同一个 App。工程上通常使用一个 Flutter Module，再在其内部用多个 Dart Package 组织业务。

---

## 19. 多 Engine 与 EngineGroup

```text
FlutterEngineGroup
  ├── Engine A → Dart Isolate A → Page A
  └── Engine B → Dart Isolate B → Page B
```

多个 Engine 适合彼此独立的 Flutter 页面实例。它们有独立 Dart Isolate 和状态，但 EngineGroup 可共享部分底层资源，降低后续 Engine 创建成本。

评估点：

- 页面是否真的需要独立状态
- Plugin 是否支持多 Engine
- Channel 是否错误绑定到全局 Messenger
- 内存和启动收益是否有实测
- Engine 销毁后原生监听是否解除

---

## 20. 性能优化

先用 DevTools、Performance Overlay 和真实设备 Profile/Release 模式测量。

构建：

- 使用 `const` Widget
- 拆小监听范围，不盲目拆小所有 Widget
- 避免在 `build` 中做 I/O、解析和大计算
- 使用 Selector 精确订阅状态

列表：

- 使用 `ListView.builder`/Sliver 懒构建
- 稳定 Key
- 合理图片尺寸与缓存
- 已知固定高度时提供 item extent
- 避免 Cell 内复杂 Intrinsic Layout

绘制：

- 谨慎使用 `Opacity`、裁剪、阴影和 SaveLayer
- 动画优先改变 transform/opacity
- `RepaintBoundary` 只用于实测存在重复绘制的区域
- 大计算使用 Isolate，但考虑复制成本

启动：

- 延迟非关键 SDK 和依赖
- Add-to-App 可预热 Engine
- 记录 Native 启动、Engine 创建、Dart Entry、首帧和可交互时间

---

## 21. 测试体系

| 测试 | 内容 |
| --- | --- |
| Unit Test | Entity、UseCase、Repository、ViewModel |
| Widget Test | 页面状态、布局和用户交互 |
| Golden Test | 关键组件视觉回归 |
| Integration Test | 真机流程与 Plugin |
| Native Test | Swift/Kotlin Channel 实现 |
| Contract Test | Dart、iOS、Android 参数与错误一致性 |

```dart
testWidgets('shows order title after loading', (tester) async {
  final repository = FakeOrderRepository(testOrder);
  await tester.pumpWidget(
    MaterialApp(home: OrderPage(repository: repository)),
  );
  await tester.pumpAndSettle();
  expect(find.text(testOrder.title), findsOneWidget);
});
```

混合工程不能只测 Dart。Channel、Engine 生命周期、原生导航和多 Engine 是高风险集成点。

---

## 22. 发布与治理

统一管理：

- Flutter SDK 版本与升级节奏
- iOS Deployment Target、Android min/target SDK
- Package Lock 和内部 Package 版本
- Flavor、签名、环境变量
- Dart Obfuscation 与符号文件
- Native/Flutter 双端日志关联
- Bundle、资源和包体积预算
- Add-to-App 灰度、回滚与兼容协议

Flutter Module 与宿主应有明确兼容矩阵，尤其是 Channel 协议、Plugin 版本和原生 SDK 版本。

---

## 23. 选型建议

适合 Flutter：

- 双端 UI 和业务共享率高
- 需要一致视觉和快速迭代
- 团队能维护 Dart 与原生边界
- 自绘 UI 的收益高于平台原生控件复用

谨慎：

- 大量深度依赖平台原生页面
- Plugin 无法满足关键 SDK
- 多窗口、系统扩展和平台特有能力占比高
- 团队没有 iOS/Android 问题定位能力

技术选型要比较总拥有成本：开发速度、性能、包体、原生集成、升级、招聘、测试和长期治理。

---

## 24. BuildContext 与 InheritedWidget

`BuildContext` 实际是 Element 的接口，表示 Widget 在树中的位置。它不是全局环境对象。

```dart
final theme = Theme.of(context);
final mediaQuery = MediaQuery.of(context);
final navigator = Navigator.of(context);
```

这些 API 沿 Element 树向上查找对应的 InheritedWidget。订阅依赖后，InheritedWidget 更新会让依赖它的 Element 重建。

常见错误：

```dart
Scaffold(
  body: TextButton(
    onPressed: () {
      // 此 context 可能位于 Scaffold 之上
      ScaffoldMessenger.of(context).showSnackBar(...);
    },
  ),
);
```

可用 `Builder` 创建位于目标节点之下的新 Context。异步间隙后使用 Context：

```dart
await viewModel.save();
if (!context.mounted) return;
Navigator.of(context).pop();
```

不要跨页面长期保存 BuildContext，它会绑定 Element 生命周期并可能导致错误导航或内存持有。

---

## 25. 自定义 InheritedWidget

```dart
class AppDependenciesScope extends InheritedWidget {
  const AppDependenciesScope({
    super.key,
    required this.dependencies,
    required super.child,
  });

  final AppDependencies dependencies;

  static AppDependencies of(BuildContext context) {
    final scope =
        context.dependOnInheritedWidgetOfExactType<AppDependenciesScope>();
    assert(scope != null, 'AppDependenciesScope not found');
    return scope!.dependencies;
  }

  @override
  bool updateShouldNotify(AppDependenciesScope oldWidget) {
    return dependencies != oldWidget.dependencies;
  }
}
```

Provider 等状态库在底层会使用类似机制。依赖对象本身不变化时，Scope 适合注入；高频变化的大状态应拆分监听粒度。

---

## 26. 状态管理方案选择

| 方案 | 特点 | 适用 |
| --- | --- | --- |
| `setState` | 局部、简单、框架原生 | 控件和页面临时状态 |
| `ValueNotifier` | 单值可监听 | 小型独立状态 |
| `ChangeNotifier` | 命令式通知 | 中小项目、MVVM |
| Provider | 依赖注入和监听封装 | 简单共享状态 |
| Riverpod | 容器化依赖、组合与测试 | 中大型项目 |
| Bloc/Cubit | Event/State 明确 | 强流程、团队规范 |
| Redux | 单 Store、Reducer | 需要严格事件追踪 |

选型问题：

1. 状态属于 Widget、页面还是整个会话？
2. 是否需要组合多个异步数据源？
3. 是否需要事件回放和严格状态机？
4. 团队能否持续遵守所选模型？
5. 测试是否可脱离 Widget Tree？

状态库不替代分层。Repository、UseCase 和领域规则不应因为使用某个状态库而进入 Widget。

---

## 27. 不可变状态与并发请求

```dart
class SearchViewModel extends ChangeNotifier {
  SearchState state = const SearchState.idle();
  int _generation = 0;

  Future<void> search(String keyword) async {
    final generation = ++_generation;
    state = const SearchState.loading();
    notifyListeners();

    try {
      final results = await repository.search(keyword);
      if (generation != _generation) return;
      state = SearchState.loaded(results);
    } catch (error) {
      if (generation != _generation) return;
      state = SearchState.failed(toMessage(error));
    }
    notifyListeners();
  }

  void cancelPendingResult() {
    _generation++;
  }
}
```

generation 防止旧请求后返回覆盖新结果，但不会终止底层工作。若 HTTP Client 支持取消，应同时取消请求。

页面状态应区分：

- 首次加载
- 下拉刷新
- 加载更多
- 空数据
- 局部操作
- 可重试与不可重试错误

不要让一个 `isLoading` 同时控制所有过程。

---

## 28. Sliver 与滚动体系

Flutter 滚动由 Scrollable、Viewport 和 Sliver 协作：

```text
ScrollView
  ↓
Scrollable：手势与滚动位置
  ↓
Viewport：可视区域
  ↓
Sliver：按滚动约束布局内容
```

```dart
CustomScrollView(
  slivers: [
    const SliverAppBar(
      pinned: true,
      expandedHeight: 200,
    ),
    SliverList.builder(
      itemCount: items.length,
      itemBuilder: (context, index) => ItemTile(item: items[index]),
    ),
  ],
);
```

- `ListView` 本质上是常用 Sliver 组合的封装
- 长列表必须使用 builder 懒构建
- 嵌套滚动需要明确联动模型，避免多个 ScrollController 抢占
- `shrinkWrap: true` 可能需要测量更多子节点，长列表中成本较高
- 分页触发应防抖并阻止重复请求

---

## 29. 动画体系

### 29.1 隐式动画

```dart
AnimatedContainer(
  duration: const Duration(milliseconds: 250),
  curve: Curves.easeOut,
  width: expanded ? 240 : 120,
  color: expanded ? Colors.blue : Colors.grey,
);
```

适合起止状态明确、不需要精细控制的动画。

### 29.2 显式动画

```dart
class FadeState extends State<Fade>
    with SingleTickerProviderStateMixin {
  late final AnimationController controller;
  late final Animation<double> opacity;

  @override
  void initState() {
    super.initState();
    controller = AnimationController(
      vsync: this,
      duration: const Duration(milliseconds: 300),
    );
    opacity = CurvedAnimation(parent: controller, curve: Curves.easeOut);
    controller.forward();
  }

  @override
  void dispose() {
    controller.dispose();
    super.dispose();
  }
}
```

`Ticker` 根据帧回调驱动 Controller，`vsync` 在不可见时避免无意义帧。动画中优先让 `AnimatedBuilder` 只重建变化子树，并把静态 child 提出。

Hero 动画要求两个路由上有相同 tag；列表中 tag 必须唯一。自定义页面转场要兼顾返回手势和系统 Reduce Motion 设置。

---

## 30. 手势与命中测试

```text
Pointer Event
  ↓ Hit Test
Gesture Recognizers
  ↓ Gesture Arena
胜出的识别器触发回调
```

多个识别器可能竞争同一指针，例如横滑、垂直滚动和点击。Gesture Arena 根据识别过程决定胜者。

```dart
GestureDetector(
  behavior: HitTestBehavior.opaque,
  onTap: onTap,
  onHorizontalDragUpdate: onDrag,
  child: child,
);
```

`AbsorbPointer` 自己参与命中但阻止子树接收；`IgnorePointer` 让子树从命中测试中消失。交互组件优先使用 Button、InkWell 等语义完整的控件。

---

## 31. Pigeon 类型安全协议

Pigeon Schema：

```dart
@ConfigurePigeon(PigeonOptions(
  dartOut: 'lib/platform/device_api.g.dart',
  swiftOut: 'ios/Classes/DeviceApi.g.swift',
  kotlinOut: 'android/src/main/kotlin/DeviceApi.g.kt',
))
class DeviceInfo {
  DeviceInfo({required this.id, required this.model});
  String id;
  String model;
}

@HostApi()
abstract class DeviceHostApi {
  @async
  DeviceInfo getDeviceInfo();
}

@FlutterApi()
abstract class DeviceFlutterApi {
  void onPermissionChanged(bool granted);
}
```

```text
HostApi：Dart 调用原生
FlutterApi：原生回调 Dart
```

生成后，Swift/Kotlin 实现强类型协议，Dart 使用生成 Client。Schema 仍需版本治理：

- 新字段尽量可选并兼容旧端
- 不随意更改已有方法语义
- 错误使用稳定 code
- 宿主和 Flutter Module 建立兼容矩阵

---

## 32. Flutter Plugin 结构

```text
device_plugin/
├── lib/
│   ├── device_plugin.dart
│   ├── device_plugin_platform_interface.dart
│   └── method_channel_device_plugin.dart
├── android/
├── ios/
├── example/
└── pubspec.yaml
```

Plugin 注册时获得 BinaryMessenger：

```swift
public final class DevicePlugin: NSObject, FlutterPlugin {
    public static func register(with registrar: FlutterPluginRegistrar) {
        let channel = FlutterMethodChannel(
            name: "device_plugin",
            binaryMessenger: registrar.messenger()
        )
        let instance = DevicePlugin()
        registrar.addMethodCallDelegate(instance, channel: channel)
    }
}
```

Android 新 Plugin API 应区分：

- `FlutterPlugin`：绑定 Engine
- `ActivityAware`：能力确实依赖 Activity
- `ServiceAware`：依赖 Service

不要在静态全局变量中保存 Activity、ViewController 或 Messenger。多 Engine 时，每个 Engine 都会有独立注册与生命周期。

---

## 33. Platform View

Platform View 用于地图、WebView、相机和复杂已有原生视图。

```dart
UiKitView(
  viewType: 'com.example/native-map',
  creationParams: {'style': 'dark'},
  creationParamsCodec: const StandardMessageCodec(),
);
```

成本与限制：

- Flutter 与原生视图合成路径更复杂
- 手势需要协调
- 透明、裁剪、变换能力存在平台差异
- 大量 Platform View 出现在滚动列表中成本较高
- 生命周期和资源释放由 Plugin 明确处理

如果只是调用平台功能，不需要原生 UI，应使用 Plugin/Channel，而不是 Platform View。

---

## 34. Add-to-App iOS 构建接入

常见集成策略：

| 方式 | 特点 |
| --- | --- |
| Xcode Build Phase | 宿主构建时同步构建 Flutter，开发方便 |
| Framework 产物 | Flutter 预构建为 XCFramework，宿主不依赖 Flutter SDK |
| Swift Package | 通过生成的 Package 接入当前 Apple 工程体系 |

团队需要决定源码集成还是二进制交付：

```text
源码集成
  优点：调试、Hot Reload 和联调方便
  成本：宿主 CI 需要 Flutter SDK，构建链耦合

二进制集成
  优点：原生团队构建稳定、职责边界清晰
  成本：每次 Dart 修改要重新发布产物，版本治理更严格
```

iOS 侧重点：

- Flutter.framework、App.framework 和 Plugin Framework 一致发布
- 真机/模拟器架构正确
- 签名、Embed 与 Strip 配置正确
- Pod/SPM 与宿主依赖无重复符号或版本冲突
- App Extension 不能直接假设复用主 App Engine

具体命令和产物形式应随项目锁定的 Flutter SDK 文档执行，避免复制其他版本脚本。

---

## 35. Add-to-App Android 构建接入

| 方式 | 特点 |
| --- | --- |
| Gradle Source Subproject | 联调方便，宿主构建 Flutter |
| AAR Repository | Flutter 预构建，宿主按 Maven Artifact 集成 |

AAR 方案通常需要为 debug/profile/release 产物建立明确版本。宿主必须选择与当前 Build Variant 匹配的产物。

Android 侧重点：

- AndroidX 与 minSdk 兼容
- ABI 拆分和 Native Library 打包
- R8/ProGuard 与 Plugin Keep Rule
- FlutterActivity、FlutterFragment 或 FlutterView 的生命周期
- Cached Engine 的首路由与 Entrypoint
- Activity Result、权限和系统返回手势

---

## 36. Engine 生命周期与预热

```text
App 启动
  ↓ 可选预热
Create FlutterEngine
  ↓
Run Dart Entrypoint
  ↓
Register Plugins
  ↓
Attach FlutterViewController / Activity
  ↓
Detach View
  ↓
复用或 Destroy Engine
```

预热降低打开首个 Flutter 页的等待，但会提前占用内存和 CPU。应测量：

- App 启动是否受影响
- Engine Ready 时间
- Flutter 首帧时间
- 常驻内存
- 页面关闭后资源回收

同一个 Engine 只能同时绑定符合平台约束的视图场景。复用时 Dart Navigator 和全局状态仍存在，宿主需要明确重置还是保留。

---

## 37. Native 与 Flutter 生命周期同步

宿主状态应通过统一 Bridge 同步：

```text
Native App Lifecycle
  ├── foreground/background
  ├── login/logout
  ├── locale/theme
  ├── network/environment
  └── memory warning
          ↓
Host Bridge
          ↓
Flutter Session Repository
          ↓
Feature ViewModels
```

登录退出不能只发送一次事件。新 Engine 或晚订阅模块还需要读取当前快照。推荐“查询当前状态 + 订阅后续变化”的组合。

---

## 38. 错误、线程与大数据通信

Platform Channel 数据经过 Codec 编解码。不要高频逐条传递传感器样本或大图片字节：

- 合并批次
- 降低采样率
- 大文件传路径或句柄
- 密集计算放原生/FFI
- UI 更新按帧节流

Channel Handler 涉及 UI API 时回到主线程；耗时 I/O 和计算不要阻塞主线程。Dart 收到结果后仍在对应 Isolate 事件循环处理。

错误协议：

```text
code: PERMISSION_DENIED
message: Camera permission denied
details:
  recoverable: true
  settingsURL: ...
```

Dart Adapter 将平台错误转换为 Domain Failure，Widget 不直接判断原生错误字符串。

---

## 39. 性能诊断方法

帧预算以设备刷新率为准。60Hz 每帧约 16.7ms，120Hz 每帧约 8.3ms。

```text
UI Thread：Dart Build / Layout / Paint
Raster Thread：图层栅格化与 GPU 提交
```

- UI 时间高：检查大范围 rebuild、同步计算、布局
- Raster 时间高：检查复杂绘制、模糊、阴影、裁剪、图片
- 两者都正常但交互慢：检查 I/O、平台线程和网络

DevTools 检查：

- Performance Timeline
- Widget Rebuild Stats
- CPU Profiler
- Memory Snapshot / Allocation
- Network
- App Size

优化必须保留前后同设备、同模式、同场景数据。Debug 模式的性能不能代表发布包。

---

## 40. 图片与内存

图片内存近似：

```text
decoded bytes ≈ width × height × 4
```

一张 4000×3000 图片解码后约占 48MB，即使压缩文件只有几 MB。

```dart
Image.network(
  url,
  cacheWidth: 800,
  fit: BoxFit.cover,
);
```

优化：

- 按显示尺寸解码
- 列表使用缩略图
- 控制内存和磁盘缓存上限
- 页面销毁后不保留大图片引用
- 预加载只覆盖高概率即将展示的资源
- 用 DevTools 验证实际峰值和泄漏

---

## 41. 无障碍、国际化与适配

```dart
Semantics(
  button: true,
  label: '提交订单',
  child: IconButton(
    onPressed: submit,
    icon: const Icon(Icons.check),
  ),
);
```

必须验证：

- 屏幕阅读器顺序和标签
- Dynamic Type / Font Scale
- 对比度和非颜色提示
- 键盘与焦点导航
- RTL 布局
- 文案扩展后的布局
- SafeArea、横竖屏、平板和分屏

不要用字符串拼接处理复数、性别和日期货币格式，应使用本地化资源与 Locale 格式化。

---

## 42. 混合工程治理清单

| 领域 | 必须明确 |
| --- | --- |
| 所有权 | Native 与 Flutter 各负责哪些页面和能力 |
| Engine | 创建、缓存、预热、销毁和多 Engine 策略 |
| 导航 | Route Schema、返回值、Deep Link、登录拦截 |
| 协议 | Pigeon/Channel 版本、兼容和错误码 |
| 生命周期 | 前后台、登录态、语言、主题、内存警告 |
| 构建 | 源码或二进制、SDK 锁定、CI、产物仓库 |
| 发布 | 灰度、回滚、宿主与 Module 兼容矩阵 |
| 监控 | Native 与 Dart Trace 串联、首帧和崩溃 |
| 测试 | Contract、Plugin、导航、多 Engine、E2E |
| 安全 | 敏感数据、日志脱敏、Channel 输入校验 |

---

## 43. 总结

| 模块 | 核心结论 |
| --- | --- |
| 渲染 | Widget 是配置，Element 管身份，RenderObject 布局绘制 |
| 状态 | 单向数据流，状态有唯一所有者 |
| 架构 | View/ViewModel、Repository/Service，复杂时增加 Domain |
| 原生交互 | Channel/Pigeon/Plugin/FFI 按场景选择 |
| 混合集成 | 宿主管 Engine、导航、生命周期和协议 |
| 模块化 | 一个 Flutter Module 内用 Dart Package 划分业务 |
| 性能 | Profile 模式测量 Build、Layout、Paint 和 Raster |
| 测试 | Dart、Channel、Native 和混合导航均需覆盖 |

## 参考资料

- [Flutter architectural overview](https://docs.flutter.dev/resources/architectural-overview)
- [Flutter app architecture guide](https://docs.flutter.dev/app-architecture/guide)
- [Platform channels](https://docs.flutter.dev/platform-integration/platform-channels)
- [Add Flutter to an existing app](https://docs.flutter.dev/add-to-app)
- [Debug an add-to-app module](https://docs.flutter.dev/add-to-app/debugging)
