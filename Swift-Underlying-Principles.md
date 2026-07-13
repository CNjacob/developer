# Swift 底层原理

## 1. Swift 语言定位

Swift 是静态类型、编译型语言，最终编译为原生机器码。它既有自己的 Runtime，也能与 Objective-C Runtime 互操作。

```text
Swift Source
  ↓
Parser / AST
  ↓
SIL
  ↓
LLVM IR
  ↓
Machine Code
```

Swift 的设计目标：

- 类型安全
- 值语义优先
- 高性能
- 现代并发
- 与 Objective-C/C/C++ 生态互操作

---

## 2. 编译流程

### 2.1 从源码到机器码

```text
.swift
  ↓ 词法/语法分析
AST
  ↓ 类型检查
Typed AST
  ↓ 生成 Swift Intermediate Language
SIL
  ↓ 优化：ARC、内联、泛型特化、去虚拟化
Optimized SIL
  ↓ 转换
LLVM IR
  ↓ 后端优化
Machine Code
```

### 2.2 SIL 的作用

SIL 是 Swift 编译器特有的中间表示，保留了 Swift 的高级语义，便于优化：

- ARC 插入与消除
- 闭包捕获分析
- 泛型特化
- 函数内联
- 去虚拟化
- 所有权检查
- async/await 状态机转换

### 2.3 编译优化示例

```swift
final class Calculator {
    func add(_ a: Int, _ b: Int) -> Int {
        a + b
    }
}
```

`final` 表示不能被继承或重写，编译器可以把动态派发优化为直接调用，甚至进一步内联。

---

## 3. 类型系统与内存布局

### 3.1 值类型和引用类型

| 类型 | 示例 | 语义 | 常见存储 |
| --- | --- | --- | --- |
| 值类型 | `struct`、`enum`、`tuple` | 赋值产生独立值 | 栈、堆、寄存器均可能 |
| 引用类型 | `class` | 赋值共享对象引用 | 堆 |

值类型不等于一定在栈上。它可能因为闭包捕获、泛型容器、存在类型装箱、类属性等原因存储在堆上。

### 3.2 Struct 内存布局

```swift
struct Point {
    var x: Double
    var y: Double
}
```

可理解为连续内存：

```text
Point
├── x: Double
└── y: Double
```

结构体方法通常不需要对象头和引用计数，性能可预测。

### 3.3 Class 内存布局

```swift
class Person {
    var age: Int = 0
}
```

类实例通常包含：

```text
Heap Object
├── metadata pointer / isa
├── reference count
└── stored properties
```

类对象由 ARC 管理生命周期。

### 3.4 Enum 内存布局

Swift `enum` 可以携带关联值：

```swift
enum ResultValue {
    case success(Data)
    case failure(Error)
}
```

底层需要存储：

- 当前 case 标记
- 关联值 payload

编译器会根据 payload 大小、额外 bit、对齐方式进行布局优化。

---

## 4. 方法调度

### 4.1 常见调度方式

| 调度方式 | 触发场景 | 性能 | 动态性 |
| --- | --- | --- | --- |
| 直接调用 | `struct`、`enum`、`final`、`private` | 最高 | 最低 |
| vtable | 普通 Swift class 方法 | 高 | 支持重写 |
| witness table | 协议泛型调用 | 中 | 支持协议抽象 |
| ObjC 消息发送 | `@objc`、`dynamic`、NSObject 子类 | 较低 | 最高 |

### 4.2 直接调用

```swift
struct Adder {
    func add(_ a: Int, _ b: Int) -> Int {
        a + b
    }
}
```

结构体方法静态可知，编译器可以直接调用、内联和优化。

### 4.3 vtable 调用

```swift
class Animal {
    func speak() {}
}

class Dog: Animal {
    override func speak() {}
}
```

类支持继承和重写，方法地址通过虚函数表查找。

### 4.4 协议见证表

```swift
protocol Drawable {
    func draw()
}

func render<T: Drawable>(_ value: T) {
    value.draw()
}
```

编译器通过 Protocol Witness Table 找到具体类型对协议方法的实现。

### 4.5 `@objc` 与 `dynamic`

```swift
@objc dynamic func reload() {}
```

- `@objc`：暴露给 Objective-C Runtime
- `dynamic`：强制动态派发，禁止编译器静态化
- `@objcMembers`：让类成员默认暴露给 ObjC，需谨慎使用

不需要 ObjC 动态能力时，应避免过度使用。

---

## 5. 泛型、`some` 与 `any`

### 5.1 泛型实现

Swift 泛型常通过类型元数据和 witness table 支持。对性能敏感路径，编译器可能进行泛型特化，生成具体类型版本。

```swift
func maxValue<T: Comparable>(_ a: T, _ b: T) -> T {
    a > b ? a : b
}
```

当调用 `maxValue(1, 2)` 时，编译器可能生成 `Int` 专用版本。

### 5.2 `some Protocol`

`some` 是不透明类型。调用方不知道具体类型，但编译器知道。

```swift
func makeView() -> some View {
    Text("Hello")
}
```

要求函数所有返回路径的底层具体类型一致。

### 5.3 `any Protocol`

`any` 是存在类型，表示“任意符合协议的值”。

```swift
let items: [any CustomStringConvertible] = [1, "A", Date()]
```

存在类型可能带来：

- 装箱
- 动态分发
- 类型信息丢失
- 不能直接使用 `Self` 或关联类型相关能力

选择建议：

- 算法和热路径优先泛型
- API 隐藏具体返回类型用 `some`
- 异构集合或插件化用 `any`

---

## 6. Copy-on-Write

Swift 标准库中的 `Array`、`Dictionary`、`Set`、`String` 使用写时复制。

```swift
var a = [1, 2, 3]
var b = a
b.append(4)

print(a) // [1, 2, 3]
print(b) // [1, 2, 3, 4]
```

复制时先共享底层 buffer，写入时如果发现 buffer 不唯一，再复制。

自定义 COW：

```swift
final class Storage {
    var values: [Int]

    init(_ values: [Int]) {
        self.values = values
    }

    func copy() -> Storage {
        Storage(values)
    }
}

struct Buffer {
    private var storage: Storage

    init(values: [Int]) {
        storage = Storage(values)
    }

    mutating func append(_ value: Int) {
        if !isKnownUniquelyReferenced(&storage) {
            storage = storage.copy()
        }
        storage.values.append(value)
    }
}
```

---

## 7. ARC 与内存管理

### 7.1 ARC 工作方式

Swift 类实例通过 ARC 管理生命周期。编译器在合适位置插入 retain/release，并在 SIL 阶段优化冗余引用计数操作。

### 7.2 强引用循环

```swift
final class Owner {
    var pet: Pet?
}

final class Pet {
    var owner: Owner?
}
```

双方强引用会导致无法释放。可用 `weak` 或 `unowned` 打破。

### 7.3 `weak` 与 `unowned`

| 修饰符 | 是否可选 | 对象释放后 | 场景 |
| --- | --- | --- | --- |
| `weak` | 必须可选 | 自动变 nil | 生命周期可能更短 |
| `unowned` | 非可选 | 访问崩溃 | 被引用对象生命周期更长 |

```swift
final class Child {
    weak var parent: Parent?
}
```

### 7.4 闭包捕获

```swift
service.load { [weak self] result in
    guard let self else { return }
    self.render(result)
}
```

不要机械地给所有闭包加 `weak`。非逃逸闭包或立即执行闭包通常不需要。

---

## 8. Swift 与 Objective-C 互操作

### 8.1 Swift 调 Objective-C

通过 Bridging Header 暴露 Objective-C 头文件：

```objc
// Project-Bridging-Header.h
#import "UserService.h"
```

Swift 中可直接使用：

```swift
let service = UserService()
```

### 8.2 Objective-C 调 Swift

Swift 类需要继承 `NSObject` 或使用 `@objc` 暴露：

```swift
@objcMembers
final class LoginService: NSObject {
    func login() {}
}
```

Objective-C 中导入：

```objc
#import "ProductModuleName-Swift.h"
```

### 8.3 桥接成本

常见桥接：

- `String` ↔ `NSString`
- `Array` ↔ `NSArray`
- `Dictionary` ↔ `NSDictionary`
- Swift closure ↔ ObjC Block

在高频路径中大量桥接可能产生性能成本。

---

## 9. Swift Concurrency

### 9.1 async/await

`async/await` 不等于创建线程。它把异步逻辑写成顺序代码，底层通过任务挂起和恢复实现。

```swift
func load() async throws -> User {
    let data = try await api.fetch()
    return try decoder.decode(User.self, from: data)
}
```

编译器会把异步函数转换为状态机。

### 9.2 Task 与结构化并发

```swift
async let profile = api.fetchProfile()
async let settings = api.fetchSettings()

let result = try await (profile, settings)
```

结构化并发强调父子任务关系，父任务取消时子任务也应被取消。

### 9.3 Actor

Actor 用于隔离可变状态：

```swift
actor Counter {
    private var value = 0

    func increment() {
        value += 1
    }

    func current() -> Int {
        value
    }
}
```

访问 actor 隔离方法通常需要 `await`。

### 9.4 Actor 重入

Actor 遇到 `await` 会让出执行权，其他任务可能进入 actor。

```swift
actor TokenStore {
    private var token: String?
    private var refreshTask: Task<String, Error>?

    func tokenValue() async throws -> String {
        if let token { return token }
        if let refreshTask { return try await refreshTask.value }

        let task = Task { try await requestToken() }
        refreshTask = task
        defer { refreshTask = nil }

        let value = try await task.value
        token = value
        return value
    }
}
```

保存进行中的 Task 可以避免重复刷新。

### 9.5 Sendable

`Sendable` 表示值可以安全跨并发域传递。

```swift
struct User: Sendable {
    let id: String
    let name: String
}
```

引用类型若使用 `@unchecked Sendable`，必须自己保证线程安全：

```swift
final class Cache: @unchecked Sendable {
    private let lock = NSLock()
    private var storage: [String: Data] = [:]
}
```

---

## 10. 性能优化要点

| 场景 | 建议 |
| --- | --- |
| 不需要继承 | 使用 `final` |
| 热路径抽象 | 优先泛型，谨慎使用 `any` |
| 大量集合修改 | 关注 COW 复制 |
| ObjC 互操作 | 避免不必要的 `@objc dynamic` |
| 闭包 | 避免不必要的逃逸和捕获 |
| 并发状态 | 使用 actor、锁或不可变设计 |
| UI 状态 | 使用 `@MainActor` 隔离 |

`@MainActor` 示例：

```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    @Published private(set) var name = ""

    func load() async {
        let profile = await service.fetchProfile()
        name = profile.name
    }
}
```

注意：CPU 密集任务不要放在主 actor 中执行。

---

## 11. 面试高频问题

1. **Swift 是静态派发还是动态派发？**  
   都有。struct/enum/final/private 常直接派发，class 走 vtable，协议泛型走 witness table，`@objc dynamic` 走动态派发。

2. **struct 一定比 class 快吗？**  
   不一定。struct 值语义更可优化，但大结构体频繁复制、COW 触发复制、存在类型装箱也会有成本。

3. **`some` 和 `any` 区别？**  
   `some` 保留具体类型，只是对外隐藏；`any` 是存在类型容器，会丢失具体类型并引入动态分发。

4. **Swift ARC 和 ObjC ARC 一样吗？**  
   原理类似，都是编译器插入引用计数操作；Swift 在 SIL 阶段有更强的所有权和优化能力。

5. **Actor 是否完全避免并发问题？**  
   Actor 能隔离状态，但要理解重入、非隔离成员、跨 actor 调用和 `Sendable` 约束。

6. **async/await 是否等于多线程？**  
   不是。它是异步任务模型，线程由运行时调度，核心是挂起和恢复。

---

## 12. 总结

Swift 的底层重点是类型系统、编译优化、方法派发、ARC、泛型、COW 和并发模型。相比 Objective-C，Swift 更倾向于静态类型和编译期优化，但在与 Objective-C 互操作、协议抽象和并发场景中仍有运行时成本。

写高质量 Swift 代码的关键是：默认使用值语义和静态派发，需要动态性时再显式引入 `class`、`any`、`@objc`、`dynamic` 等能力。
