# Dart 语言核心知识

## 1. Dart 的定位与执行方式

Dart 是强类型、支持类型推断和空安全的面向对象语言，也是 Flutter 的主要开发语言。

```text
开发阶段：Dart → JIT / VM → Hot Reload
发布阶段：Dart → AOT Native Code → iOS / Android / Desktop
Web：     Dart → JavaScript / WebAssembly
```

- JIT 适合开发期快速编译和热重载
- AOT 适合发布包的启动速度和稳定性能
- Dart 中除 `null` 外的值都属于对象
- 类型检查主要发生在编译期，泛型信息在运行时仍可用

---

## 2. 变量、常量与类型推断

```dart
var name = 'Ada';           // 推断为 String
String city = 'Shanghai';   // 显式类型
final createdAt = DateTime.now();
const maxRetryCount = 3;
Object value = 'text';
dynamic unchecked = fetchValue();
```

| 关键字/类型 | 含义 |
| --- | --- |
| `var` | 根据初始值推断静态类型 |
| `final` | 只能赋值一次，值可在运行时确定 |
| `const` | 编译期常量，创建规范化常量对象 |
| `Object` | 可接收任意非空对象，使用前仍受类型检查 |
| `Object?` | 可接收任意值，包括 `null` |
| `dynamic` | 跳过大部分静态检查，应限制在边界 |

```dart
final list = <int>[1, 2];
list.add(3);                // 合法：引用不变，对象可变

const immutable = <int>[1, 2];
// immutable.add(3);        // 运行时不允许修改
```

默认使用 `final`，需要重新绑定时使用 `var`，真正的编译期常量使用 `const`。

---

## 3. 空安全

Dart 类型默认不可空，只有带 `?` 的类型能接收 `null`。

```dart
String name = 'Ada';
String? nickname;

final length = nickname?.length;       // int?
final display = nickname ?? name;      // 空值回退
nickname ??= 'Unknown';                // 为空时赋值
final forced = nickname!.length;       // 非空断言，误判会崩溃
```

`late` 表示稍后初始化：

```dart
class Controller {
  late final Repository repository;

  void configure(Repository value) {
    repository = value;
  }
}
```

`late` 不是绕过空安全的默认工具。读取前未初始化会产生运行时错误，构造函数注入通常更安全。

类型提升：

```dart
String label(String? value) {
  if (value == null) return 'Empty';
  return value.trim(); // 已提升为 String
}
```

字段可能被其他代码修改时，编译器不一定能提升；可先复制到局部 `final` 变量。

---

## 4. 内置集合

### 4.1 List、Set、Map

```dart
final numbers = <int>[1, 2, 3];
final unique = <String>{'A', 'B'};
final scores = <String, int>{'Ada': 100};

final doubled = numbers.map((value) => value * 2).toList();
final evens = numbers.where((value) => value.isEven).toList();
final sum = numbers.fold(0, (total, value) => total + value);
```

集合字面量支持条件和展开：

```dart
final actions = <String>[
  'view',
  if (canEdit) 'edit',
  if (extraActions != null) ...extraActions,
];
```

注意：

- `map`、`where` 通常返回惰性 `Iterable`
- 多次遍历会重复计算，需要快照时调用 `toList`
- `List.unmodifiable` 提供只读视图语义
- 自定义对象作为 Map Key 或 Set 元素时，应正确实现 `==` 与 `hashCode`

---

## 5. 函数

```dart
int add(int left, int right) => left + right;

String greet(
  String name, {
  String prefix = 'Hello',
  required bool excited,
}) {
  return '$prefix, $name${excited ? '!' : ''}';
}
```

Dart 支持位置参数、可选位置参数和命名参数：

```dart
void position(int x, int y, [int z = 0]) {}
void request(String path, {Duration? timeout, required String method}) {}
```

函数是一等对象：

```dart
typedef Predicate<T> = bool Function(T value);

List<T> filter<T>(Iterable<T> values, Predicate<T> predicate) {
  return values.where(predicate).toList();
}
```

闭包捕获词法作用域：

```dart
int Function() createCounter() {
  var count = 0;
  return () => ++count;
}
```

---

## 6. 类、构造函数与不可变对象

```dart
class User {
  const User({
    required this.id,
    required this.name,
    this.nickname,
  });

  final String id;
  final String name;
  final String? nickname;

  String get displayName => nickname ?? name;

  User copyWith({String? name, String? nickname}) {
    return User(
      id: id,
      name: name ?? this.name,
      nickname: nickname ?? this.nickname,
    );
  }

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is User &&
          other.id == id &&
          other.name == name &&
          other.nickname == nickname;

  @override
  int get hashCode => Object.hash(id, name, nickname);
}
```

构造方式：

```dart
class Point {
  const Point(this.x, this.y);
  const Point.origin() : x = 0, y = 0; // 命名构造

  factory Point.fromJson(Map<String, Object?> json) {
    return Point(json['x'] as double, json['y'] as double);
  }

  final double x;
  final double y;
}
```

- 生成式构造函数总是创建实例
- `factory` 可返回缓存、子类型或执行解析
- `const` 构造要求字段为 `final`
- Flutter Widget 应尽可能不可变并提供 `const` 构造

---

## 7. 继承、接口、Mixin 与扩展

Dart 每个类都隐式定义接口：

```dart
abstract interface class UserRepository {
  Future<User> find(String id);
}

final class RemoteUserRepository implements UserRepository {
  @override
  Future<User> find(String id) async {
    // ...
  }
}
```

现代类修饰符：

| 修饰符 | 主要约束 |
| --- | --- |
| `abstract` | 不能直接实例化 |
| `interface` | 库外只能实现，不能继承 |
| `base` | 库外只能继承，保持实现继承关系 |
| `final` | 库外不能继承或实现 |
| `sealed` | 子类型限制在同一库，利于穷尽检查 |
| `mixin` | 声明可复用行为 |

Mixin：

```dart
mixin Loggable {
  void log(String message) => print(message);
}

class CheckoutService with Loggable {}
```

Extension：

```dart
extension StringValidation on String {
  bool get isValidEmail => contains('@') && contains('.');
}
```

优先组合而非深继承。Extension 适合为已有类型补充无状态工具，不会真正修改原类型。

---

## 8. 泛型

```dart
abstract interface class Repository<ID, Entity> {
  Future<Entity> find(ID id);
  Future<void> save(Entity entity);
}

T firstOr<T>(List<T> values, T fallback) {
  return values.isEmpty ? fallback : values.first;
}
```

泛型约束：

```dart
num total<T extends num>(Iterable<T> values) {
  return values.fold<num>(0, (sum, value) => sum + value);
}
```

Dart 泛型具有运行时信息，`names is List<String>` 可以执行真实检查。设计泛型时应表达类型关系，不要只为减少几行重复代码而引入难懂抽象。

---

## 9. Records 与 Patterns

Record 适合表达轻量、固定结构的多返回值：

```dart
(User, bool) loadCachedUser() => (user, true);

({User user, bool fromCache}) loadUser() {
  return (user: user, fromCache: true);
}

final (:user, :fromCache) = loadUser();
```

Pattern 用于解构与匹配：

```dart
String describe(Object value) => switch (value) {
  int value when value > 0 => 'positive $value',
  int value => 'integer $value',
  String(:final length) => 'string length: $length',
  [final first, ...final rest] => '$first + ${rest.length}',
  _ => 'unknown',
};
```

Sealed Class 配合穷尽匹配：

```dart
sealed class LoadState<T> {
  const LoadState();
}

final class Loading<T> extends LoadState<T> {}
final class Loaded<T> extends LoadState<T> {
  const Loaded(this.data);
  final T data;
}
final class Failed<T> extends LoadState<T> {
  const Failed(this.error);
  final Object error;
}

Widget buildState(LoadState<User> state) => switch (state) {
  Loading() => const CircularProgressIndicator(),
  Loaded(:final data) => Text(data.name),
  Failed(:final error) => Text('$error'),
};
```

Records 适合局部结构；跨模块公共模型通常仍用命名类，便于扩展、文档和序列化。

---

## 10. 异常与 Result 建模

```dart
Future<User> loadUser(String id) async {
  try {
    return await repository.find(id);
  } on TimeoutException catch (error, stackTrace) {
    logger.error(error, stackTrace);
    throw const UserFailure.timeout();
  } catch (error, stackTrace) {
    Error.throwWithStackTrace(
      UserFailure.unknown(error),
      stackTrace,
    );
  } finally {
    metrics.finish();
  }
}
```

- `throw` 可抛出任意对象，但应统一使用异常类型
- `rethrow` 保留原始堆栈
- `Error.throwWithStackTrace` 可在转换错误时保留堆栈
- 可预期的业务失败可以使用 sealed Result 类型显式建模
- 编程错误不应全部吞掉并转换成“失败提示”

---

## 11. Future、Stream 与异步控制

`Future<T>` 表示一个异步结果：

```dart
Future<(User, List<Permission>)> loadPage() async {
  final results = await Future.wait([
    userRepository.current(),
    permissionRepository.all(),
  ]);
  return (results[0] as User, results[1] as List<Permission>);
}
```

独立任务应并发启动；有数据依赖的任务再串行等待。

`Stream<T>` 表示多个异步事件：

```dart
Stream<int> countdown(int from) async* {
  for (var value = from; value >= 0; value--) {
    yield value;
    await Future<void>.delayed(const Duration(seconds: 1));
  }
}

final subscription = countdown(3).listen(print);
await subscription.cancel();
```

需要明确：

- Single-subscription Stream 只能有一个监听者
- Broadcast Stream 允许多个监听者，但通常不缓存旧事件
- 页面销毁时取消 Subscription
- Dart `Future` 本身没有通用强制取消；取消需要底层 API、Operation 或请求标识协作

---

## 12. Event Loop 与微任务

```text
同步调用栈
   ↓ 清空
Microtask Queue
   ↓ 清空
Event Queue：Timer、I/O、UI Event
```

```dart
void main() {
  print('A');
  Future(() => print('B'));
  scheduleMicrotask(() => print('C'));
  Future.microtask(() => print('D'));
  print('E');
}
// A E C D B
```

微任务会优先于事件队列，但大量递归微任务可能饿死 UI 和 I/O。业务代码通常使用 `Future`/`async` 即可，不应滥用 `scheduleMicrotask`。

---

## 13. Isolate 并发模型

每个 Isolate 有独立堆和事件循环，通过消息传递通信，不共享普通可变对象。

```text
Main Isolate  ← SendPort / ReceivePort → Worker Isolate
```

短时计算：

```dart
Map<String, Object?> parsePayload(String source) {
  return jsonDecode(source) as Map<String, Object?>;
}

final data = await Isolate.run(() => parsePayload(largeJson));
```

长期 Worker：

```dart
void worker(SendPort parentPort) {
  final receivePort = ReceivePort();
  parentPort.send(receivePort.sendPort);
  receivePort.listen((message) {
    // 处理消息并通过 parentPort 返回
  });
}
```

适合 CPU 密集型 JSON、图片、压缩或加密计算。网络等待本身不需要 Isolate。创建和传输也有成本，小任务放入 Isolate 可能更慢。

---

## 14. Library、Package 与可见性

下划线开头的声明在 Library 内私有：

```dart
class _InternalClient {}
```

公共模块通过入口文件控制 API：

```dart
// lib/account.dart
export 'src/account.dart' show Account;
export 'src/account_service.dart' show AccountService;
```

```text
account_package/
├── lib/
│   ├── account.dart
│   └── src/
├── test/
├── example/
└── pubspec.yaml
```

- 外部只依赖 `package:account/account.dart`
- `src` 保存内部实现
- 使用 `dependency_overrides` 只做本地开发，不长期提交不可控覆盖
- Package API 应遵循语义化版本和弃用周期

---

## 15. JSON 与数据边界

```dart
class UserDto {
  const UserDto({required this.id, required this.displayName});

  factory UserDto.fromJson(Map<String, Object?> json) {
    final id = json['id'];
    final displayName = json['display_name'];
    if (id is! String || displayName is! String) {
      throw const FormatException('Invalid user');
    }
    return UserDto(id: id, displayName: displayName);
  }

  final String id;
  final String displayName;

  User toDomain() => User(id: id, name: displayName);
}
```

不要让 `Map<String, dynamic>` 穿透整个项目。应在网络、缓存和 Platform Channel 边界立刻校验并转换为类型模型。大型项目可使用代码生成减少机械序列化代码，但生成代码不能代替协议设计。

---

## 16. 测试示例

```dart
abstract interface class Clock {
  DateTime now();
}

class CouponService {
  CouponService(this.clock);
  final Clock clock;

  bool isExpired(DateTime expireAt) => !expireAt.isAfter(clock.now());
}

class FixedClock implements Clock {
  FixedClock(this.value);
  final DateTime value;

  @override
  DateTime now() => value;
}
```

将时间、随机数、网络和存储作为依赖注入，业务测试无需依赖真实环境。

---

## 17. 常见误区

- 用 `dynamic` 快速消除类型错误
- 到处使用 `!` 和 `late`
- 在 `build` 或 getter 中执行有副作用的操作
- 忽略 `StreamSubscription`、Timer 和 Controller 的释放
- 为简单数据创建过度复杂的继承树
- 把 DTO、Domain Model 和 UI Model 混为一个类型
- 对等待网络的任务滥用 Isolate
- 公共 Package 导出所有内部实现

---

## 18. 运算符与常用语法

### 18.1 条件成员访问与级联

```dart
final city = user?.address?.city ?? 'Unknown';

final paint = Paint()
  ..color = Colors.blue
  ..strokeWidth = 2
  ..style = PaintingStyle.stroke;
```

`..` 对同一个对象连续操作，表达式返回被操作对象；`?..` 用于第一个可能为空的接收者：

```dart
querySelector('#confirm')
  ?..text = 'Confirm'
  ..classes.add('primary');
```

级联不是链式调用。链式调用把上一个方法返回值传给下一个方法；级联始终操作最初的接收对象。

### 18.2 类型判断与转换

```dart
if (value is String) {
  print(value.length); // 自动提升
}

final user = value as User;   // 不符合时抛出 TypeError
final userOrNull = value is User ? value : null;
```

`as` 不是数据转换，只是运行时类型断言。JSON 的 Map 不能通过 `as User` 变成 User，必须显式解析。

### 18.3 相等与身份

```dart
identical(first, second); // 是否同一对象
first == second;          // 可重载的值相等
```

重载 `==` 时必须同步实现 `hashCode`，并保持：

- 自反：`a == a`
- 对称：`a == b` 与 `b == a` 一致
- 传递：`a == b && b == c` 则 `a == c`
- 相等对象拥有相同 `hashCode`

---

## 19. 流程控制与 Pattern

```dart
for (final entry in scores.entries) {
  print('${entry.key}: ${entry.value}');
}

for (var index = 0; index < values.length; index++) {
  // 需要索引时使用
}

while (queue.isNotEmpty) {
  process(queue.removeFirst());
}
```

带标签的跳转应少用，但处理嵌套循环时有明确含义：

```dart
outer:
for (final row in matrix) {
  for (final value in row) {
    if (value < 0) break outer;
  }
}
```

`switch` 表达式适合返回值：

```dart
String statusText(OrderStatus status) => switch (status) {
  OrderStatus.pending => '待支付',
  OrderStatus.paid => '已支付',
  OrderStatus.cancelled => '已取消',
};
```

对象 Pattern：

```dart
String userLabel(User user) => switch (user) {
  User(name: final name, age: final age) when age >= 18 => '$name（成人）',
  User(name: final name) => '$name（未成年）',
};
```

穷尽 `switch` 能在增加枚举值或 sealed 子类型时产生编译错误，比 `default` 静默吞掉新分支更安全。

---

## 20. 构造函数深入

### 20.1 初始化列表

```dart
class Temperature {
  Temperature.celsius(this.celsius)
      : assert(celsius >= -273.15);

  Temperature.fahrenheit(double value)
      : celsius = (value - 32) * 5 / 9;

  final double celsius;
}
```

初始化顺序：

1. 初始化列表
2. 父类构造函数
3. 当前类构造函数体

字段必须在构造函数体执行前完成初始化。

### 20.2 重定向与 Factory

```dart
class ApiClient {
  ApiClient._(this.baseUrl);

  factory ApiClient.production() =>
      ApiClient._('https://api.example.com');

  factory ApiClient.environment(Environment environment) => switch (environment) {
    Environment.production => ApiClient.production(),
    Environment.staging => ApiClient._('https://staging.example.com'),
  };

  final String baseUrl;
}
```

`factory` 中没有实例 `this`，可以返回子类或缓存实例。不要用 Factory 隐藏大量 I/O；异步创建更适合命名静态方法：

```dart
static Future<Database> open(String path) async {
  final handle = await NativeDatabase.open(path);
  return Database._(handle);
}
```

---

## 21. 抽象类、接口与组合

接口隔离：

```dart
abstract interface class Reader<T> {
  Future<T> read();
}

abstract interface class Writer<T> {
  Future<void> write(T value);
}

class UserStore implements Reader<User>, Writer<User> {
  // ...
}
```

调用方只依赖需要的最小接口，测试 Fake 也更容易编写。

组合：

```dart
class RetryingUserRepository implements UserRepository {
  RetryingUserRepository(this.decorated, this.policy);

  final UserRepository decorated;
  final RetryPolicy policy;

  @override
  Future<User> find(String id) {
    return policy.execute(() => decorated.find(id));
  }
}
```

装饰器可以组合缓存、重试、埋点，而不让一个 Repository 同时承担所有横切职责。

---

## 22. Extension Type

Extension Type 为现有表示类型提供不同的静态接口，适合业务 ID：

```dart
extension type const UserId(String value) {
  bool get isValid => value.isNotEmpty;
}

extension type const OrderId(String value) {}

Future<User> loadUser(UserId id) async {
  // ...
}
```

`UserId` 与 `OrderId` 在静态类型上不同，可以减少参数传错。它们通常没有额外运行时包装成本，但也不等同于安全校验；从外部数据创建时仍需检查格式。

---

## 23. 泛型边界与协变

Dart 类泛型通常是协变的，例如 `List<Dog>` 可视为 `List<Animal>`，但写入不兼容值会被运行时检查阻止。

```dart
final List<Dog> dogs = [Dog()];
final List<Animal> animals = dogs;
// animals.add(Cat()); // 运行时类型错误
```

API 设计中，读取集合优先暴露 `Iterable<T>`，不要暴露可变 `List<T>`：

```dart
class Zoo {
  final List<Animal> _animals = [];
  Iterable<Animal> get animals => _animals;

  void add(Animal animal) => _animals.add(animal);
}
```

方法参数可显式使用 `covariant` 缩窄覆盖类型，但这会把部分安全检查推迟到运行时，应谨慎使用。

---

## 24. Future 调度与错误传播

```dart
Future<void> example() async {
  try {
    final user = await loadUser();
    await saveUser(user);
  } catch (error, stackTrace) {
    logger.record(error, stackTrace);
  }
}
```

常见错误：

```dart
items.forEach((item) async {
  await save(item);
});
// forEach 不等待回调
```

串行：

```dart
for (final item in items) {
  await save(item);
}
```

并行：

```dart
await Future.wait(items.map(save));
```

忽略 Future 必须显式表达，并确保错误被处理：

```dart
unawaited(
  analytics.track(event).catchError(logger.record),
);
```

超时不会自动终止底层任务：

```dart
await request.timeout(const Duration(seconds: 10));
```

它只让等待方收到 `TimeoutException`。底层网络、文件或原生任务要使用自身的取消能力。

---

## 25. Stream 深入

Stream 转换：

```dart
final validUsers = userEvents
    .where((user) => user.isActive)
    .map((user) => user.name)
    .distinct();
```

Controller：

```dart
class SessionRepository {
  final _controller = StreamController<Session?>.broadcast();

  Stream<Session?> get sessions => _controller.stream;

  void update(Session? session) => _controller.add(session);

  Future<void> dispose() => _controller.close();
}
```

广播流适合多个监听者，但订阅较晚的监听者收不到历史事件。若订阅者需要当前状态，应同时暴露同步快照，或使用具有 replay/value 语义的状态容器。

Stream 错误是事件，不一定自动关闭流：

```dart
subscription = stream.listen(
  onData,
  onError: onError,
  onDone: onDone,
  cancelOnError: false,
);
```

背压需要业务设计。生产速度远高于消费速度时，应丢弃、合并、采样或在数据源侧限速。

---

## 26. Zone 与全局错误

Zone 提供异步执行上下文，可用于统一捕获未处理异步错误和传递请求上下文。

```dart
void main() {
  runZonedGuarded(
    () => runApp(const App()),
    (error, stackTrace) {
      crashReporter.record(error, stackTrace);
    },
  );
}
```

Flutter 还需要分别处理框架错误和平台分发错误。Zone 不应该被用来隐藏正常的业务错误；业务错误仍应在 Repository/ViewModel 边界显式处理。

---

## 27. 资源所有权与内存

Dart 使用垃圾回收，但 GC 只管理内存，不会自动处理所有外部资源。

必须显式释放：

- `StreamSubscription.cancel`
- `StreamController.close`
- `Timer.cancel`
- Socket、文件和数据库句柄
- Flutter Controller、FocusNode
- FFI 分配的 Native Memory

```dart
class ResourceScope {
  StreamSubscription<Object?>? subscription;
  Timer? timer;

  Future<void> dispose() async {
    timer?.cancel();
    await subscription?.cancel();
  }
}
```

常见泄漏是长生命周期单例持有页面回调、广播流监听未取消、缓存无限增长，以及原生回调仍引用 Dart 对象。

---

## 28. Lint 与工程规范

`analysis_options.yaml`：

```yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  language:
    strict-casts: true
    strict-inference: true
    strict-raw-types: true

linter:
  rules:
    avoid_dynamic_calls: true
    cancel_subscriptions: true
    close_sinks: true
    discarded_futures: true
    unawaited_futures: true
```

严格规则应结合项目逐步开启，目标是让边界和生命周期错误在代码评审前暴露。

---

## 29. 总结

| 模块 | 核心结论 |
| --- | --- |
| 类型 | 优先静态类型、`final` 和完整空安全 |
| 对象 | 默认不可变，组合优先于深继承 |
| 建模 | Sealed Class + Pattern 表达有限状态 |
| 异步 | Future 表达单结果，Stream 表达事件序列 |
| 并发 | Isolate 不共享堆，通过消息传递 |
| 边界 | JSON 和平台数据进入系统时立即校验 |
| 模块 | 用 Package 入口控制稳定公共 API |

## 参考资料

- [Dart Language](https://dart.dev/language)
- [Records](https://dart.dev/language/records)
- [Patterns](https://dart.dev/language/patterns)
- [Concurrency](https://dart.dev/language/concurrency)
