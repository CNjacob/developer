# Swift 语言知识与底层原理

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

### 1.1 常见英文缩写

| 缩写 | 全称 | 作用 |
| --- | --- | --- |
| ARC | Automatic Reference Counting | 自动引用计数，管理 class 实例生命周期 |
| COW | Copy-on-Write | 写时复制，减少值类型集合复制成本 |
| ABI | Application Binary Interface | 应用二进制接口，影响模块、库和系统之间的二进制兼容 |
| API | Application Programming Interface | 应用编程接口，代码对外暴露的调用契约 |
| SIL | Swift Intermediate Language | Swift 中间表示，用于 ARC、泛型、async 等优化 |
| LLVM IR | LLVM Intermediate Representation | LLVM 中间表示，继续优化并生成机器码 |
| PWT | Protocol Witness Table | 协议见证表，记录具体类型如何实现协议方法 |
| POP | Protocol-Oriented Programming | 面向协议编程，用协议、扩展、泛型组合能力 |
| FP | Functional Programming | 函数式编程，用函数和值转换组织逻辑，减少共享可变状态 |
| GCD | Grand Central Dispatch | Apple 的任务调度系统，基于队列和线程池执行任务 |
| QoS | Quality of Service | 任务优先级/服务质量，影响系统调度资源分配 |
| UI | User Interface | 用户界面，iOS 中必须在主线程更新 |
| SDK | Software Development Kit | 软件开发工具包，包含 API、工具和文档 |
| ObjC | Objective-C | Apple 平台上常与 Swift 互操作的动态语言 |

---

## 2. Swift 语法基础

### 2.1 变量、常量和类型推断

`let` 定义常量，赋值后不能修改；`var` 定义变量，可以再次赋值。Swift 支持类型推断，但公共 API、复杂闭包和容易误读的地方建议显式标注类型。

```swift
let appName = "iOSer"          // String，常量
var retryCount = 0             // Int，变量
let timeout: TimeInterval = 5  // 显式标注，TimeInterval 本质是 Double

retryCount += 1
```

常见建议：

- 默认使用 `let`，需要改变时再改成 `var`
- 金额、坐标、时间等业务含义强的值，优先显式写类型
- 不要为了省代码让表达式类型难以判断

### 2.2 基本类型

| 类型 | 示例 | 说明 |
| --- | --- | --- |
| `Int` / `UInt` | `42` | 有符号/无符号整数，iOS 64 位设备上通常为 64 位 |
| `Double` / `Float` | `3.14` | 浮点数，默认推断为 `Double` |
| `Bool` | `true` | 只能是 `true` 或 `false`，不能用 `0/1` 替代 |
| `String` | `"Hello"` | Unicode 字符串，按字符处理可能不是 O(1) |
| `Character` | `"A"` | 单个字符，可能由多个 Unicode 标量组成 |
| `Optional<T>` | `String?` | 可能有值也可能为 `nil` |
| `Tuple` | `(id: 1, name: "A")` | 临时组合多个值，适合局部返回 |

```swift
let isLoggedIn: Bool = true
let score: Double = 98.5
let user: (id: Int, name: String) = (id: 1001, name: "Lee")

print(user.id)   // 1001
print(user.name) // Lee
```

### 2.3 Optional 可选值

`Optional` 表示值可能不存在。`String?` 等价于 `Optional<String>`。

```swift
let text: String? = "42"

// if let：有值时进入分支
if let value = text {
    print(value.count)
}

// guard let：提前退出，减少嵌套
func parseAge(_ input: String?) -> Int? {
    guard let input else { return nil }
    return Int(input)
}

// nil 合并：没有值时使用默认值
let displayName = text ?? "Unknown"
```

不要滥用强制解包 `!`：

```swift
let value = Int("abc")
// print(value!) // 崩溃：value 是 nil
```

适合使用 `!` 的场景通常很少，例如 IBOutlet 生命周期由 storyboard 保证，或测试中明确希望失败时崩溃。

### 2.4 控制流

Swift 的 `if` 条件必须是 `Bool`。`switch` 必须覆盖所有情况，适合处理枚举、范围和模式匹配。

```swift
let score = 86

switch score {
case 90...100:
    print("A")
case 80..<90:
    print("B")
case 60..<80:
    print("C")
default:
    print("D")
}
```

`for-in` 常用于遍历区间和集合：

```swift
for index in 0..<3 {
    print(index) // 0, 1, 2
}

let names = ["Ava", "Ben"]
for (index, name) in names.enumerated() {
    print("\(index): \(name)")
}
```

### 2.5 函数、参数标签和返回值

Swift 函数参数默认有外部参数名。第一个参数默认没有外部名，后续参数默认有外部名。

```swift
func makeUser(id: Int, name: String) -> String {
    "\(id)-\(name)"
}

let user = makeUser(id: 1, name: "Lee")
```

可以用 `_` 省略外部参数名：

```swift
func add(_ a: Int, _ b: Int) -> Int {
    a + b
}

let result = add(1, 2)
```

`inout` 表示函数可以修改传入变量：

```swift
func increase(_ value: inout Int) {
    value += 1
}

var count = 0
increase(&count) // & 表示传入可被修改的引用位置
```

### 2.6 闭包

闭包是可以被传递和保存的代码块，常用于回调、集合转换、异步任务。

```swift
let numbers = [1, 2, 3]

let squares = numbers.map { value in
    value * value
}

let evenNumbers = numbers.filter { $0.isMultiple(of: 2) }
```

闭包捕获外部对象时要注意生命周期：

```swift
final class ViewModel {
    private let service = UserService()

    func load() {
        service.fetch { [weak self] result in
            // weak 避免 service 长时间持有闭包时造成循环引用
            guard let self else { return }
            self.handle(result)
        }
    }

    private func handle(_ result: Result<User, Error>) {}
}
```

### 2.7 struct、class 和 enum

| 类型 | 语义 | 适合场景 |
| --- | --- | --- |
| `struct` | 值类型，赋值通常表示独立值 | Model、配置、坐标、状态 |
| `class` | 引用类型，赋值共享同一对象 | 需要身份、继承、共享可变状态 |
| `enum` | 枚举类型，可携带关联值 | 有限状态、结果、路由、错误 |

```swift
struct User {
    let id: Int
    var name: String
}

final class Session {
    var token: String?
}

enum LoadState {
    case idle
    case loading
    case success([User])
    case failure(Error)
}
```

`enum` 搭配 `switch` 能清晰表达状态：

```swift
func render(_ state: LoadState) {
    switch state {
    case .idle:
        print("等待加载")
    case .loading:
        print("加载中")
    case .success(let users):
        print("用户数：\(users.count)")
    case .failure(let error):
        print("错误：\(error.localizedDescription)")
    }
}
```

### 2.8 协议和扩展

`protocol` 定义能力，具体类型负责实现。`extension` 可以给已有类型增加方法，也可以提供协议默认实现。

```swift
protocol Trackable {
    var eventName: String { get }
    func track()
}

extension Trackable {
    func track() {
        print("track: \(eventName)")
    }
}

struct LoginEvent: Trackable {
    let eventName = "login"
}

LoginEvent().track()
```

协议常用于：

- 定义模块边界
- 做依赖注入和单元测试替身
- 为多个类型抽象共同能力
- 配合泛型保留静态类型信息

### 2.9 属性和访问控制

常见访问控制：

| 修饰符 | 作用范围 |
| --- | --- |
| `private` | 当前声明作用域内 |
| `fileprivate` | 当前文件内 |
| `internal` | 当前模块内，默认级别 |
| `public` | 模块外可访问，但模块外不可继承/重写 |
| `open` | 模块外可访问，也允许继承/重写 |

计算属性示例：

```swift
struct Rectangle {
    var width: Double
    var height: Double

    var area: Double {
        width * height // 只读计算属性
    }
}
```

属性观察器示例：

```swift
struct Volume {
    var value: Int = 0 {
        didSet {
            // didSet 在属性设置后调用，可用于限制范围或触发副作用
            value = min(max(value, 0), 100)
        }
    }
}
```

---

## 3. Swift 集合类型

Swift 标准库最常用的集合是 `Array`、`Dictionary`、`Set`。它们都是泛型集合，也是值类型，并采用 COW 优化复制成本。

| 集合 | 形式 | 特点 | 常见场景 |
| --- | --- | --- | --- |
| `Array<Element>` | `[Element]` | 有序、可重复、通过下标访问 | 列表数据、接口返回顺序、表格数据源 |
| `Dictionary<Key, Value>` | `[Key: Value]` | 键值对、Key 唯一、Key 必须 `Hashable` | 按 id 查询、缓存、参数表 |
| `Set<Element>` | `Set<Element>` | 无序、不重复、Element 必须 `Hashable` | 去重、交集/并集、快速判断是否存在 |

### 3.1 Array

```swift
var names: [String] = ["Ava", "Ben"]

names.append("Cindy")        // 追加
names.insert("Dan", at: 0)   // 指定位置插入
names.remove(at: 1)          // 删除指定位置

let first = names.first      // Optional<String>
let count = names.count
let containsAva = names.contains("Ava")
```

安全访问下标时要先判断范围：

```swift
let index = 2
if names.indices.contains(index) {
    print(names[index])
}
```

常见操作：

```swift
let numbers = [1, 2, 3, 4, 5]

let evens = numbers.filter { $0.isMultiple(of: 2) } // 筛选：[2, 4]
let doubled = numbers.map { $0 * 2 }                // 转换：[2, 4, 6, 8, 10]
let sum = numbers.reduce(0) { partial, value in     // 聚合：15
    partial + value
}

let sorted = numbers.sorted(by: >)                  // 排序：[5, 4, 3, 2, 1]
```

### 3.2 Dictionary

```swift
var scores: [String: Int] = [
    "Ava": 95,
    "Ben": 82
]

scores["Cindy"] = 90          // 新增
scores["Ben"] = 88            // 修改
scores["Ava"] = nil           // 删除

let benScore = scores["Ben"]  // Optional<Int>
let defaultScore = scores["Unknown", default: 0]
```

遍历和转换：

```swift
for (name, score) in scores {
    print("\(name): \(score)")
}

let passedNames = scores
    .filter { _, score in score >= 60 }
    .map { name, _ in name }
```

把数组按 id 转成字典，便于 O(1) 查询：

```swift
struct User: Hashable {
    let id: Int
    let name: String
}

let users = [
    User(id: 1, name: "Ava"),
    User(id: 2, name: "Ben")
]

let userByID = Dictionary(uniqueKeysWithValues: users.map { ($0.id, $0) })
let user = userByID[2]
```

如果 key 可能重复，使用分组：

```swift
let grouped = Dictionary(grouping: users) { user in
    user.name.first ?? "#"
}
```

### 3.3 Set

```swift
var selectedIDs: Set<Int> = [1, 2, 3]

selectedIDs.insert(4)
selectedIDs.remove(2)

if selectedIDs.contains(3) {
    print("已选择")
}
```

集合运算：

```swift
let a: Set = [1, 2, 3]
let b: Set = [3, 4, 5]

let union = a.union(b)                    // 并集：[1, 2, 3, 4, 5]
let intersection = a.intersection(b)      // 交集：[3]
let subtracting = a.subtracting(b)        // 差集：[1, 2]
let symmetric = a.symmetricDifference(b)  // 对称差集：[1, 2, 4, 5]
```

### 3.4 集合常见使用场景

| 场景 | 推荐做法 |
| --- | --- |
| 展示有序列表 | `Array` 作为数据源 |
| 按 id 快速查找对象 | `Dictionary<ID, Model>` |
| 判断某个 id 是否已选中 | `Set<ID>` |
| 接口数组去重 | 转成 `Set` 或用字典按 id 合并 |
| 多条件筛选 | `filter`，复杂场景拆成命名谓词 |
| 批量字段转换 | `map` / `compactMap` |
| 扁平化二维数组 | `flatMap` |
| 统计总数、金额 | `reduce` |
| 按首字母、日期分组 | `Dictionary(grouping:by:)` |
| 列表 diff 前建立索引 | `Dictionary(uniqueKeysWithValues:)` |

示例：接口返回用户列表，去重、排序、分组：

```swift
struct Contact: Hashable {
    let id: Int
    let name: String
    let isFriend: Bool
}

let response = [
    Contact(id: 2, name: "Ben", isFriend: true),
    Contact(id: 1, name: "Ava", isFriend: false),
    Contact(id: 2, name: "Ben", isFriend: true)
]

// 1. 用 Dictionary 按 id 去重。重复 key 时保留新值。
let uniqueByID = Dictionary(
    response.map { ($0.id, $0) },
    uniquingKeysWith: { _, new in new }
)

// 2. 取出 value 后排序，保证 UI 展示顺序稳定。
let sortedContacts = uniqueByID.values.sorted { $0.name < $1.name }

// 3. 按是否好友分组。
let grouped = Dictionary(grouping: sortedContacts) { contact in
    contact.isFriend ? "friends" : "others"
}
```

### 3.5 集合性能注意点

- `Array` 下标访问快，但在头部插入/删除需要移动后续元素
- `Dictionary` 和 `Set` 查询通常很快，但元素必须正确实现 `Hashable`
- `map`、`filter`、`sorted` 会产生新集合，大数据热路径要关注临时分配
- 值类型集合赋值通常先共享 buffer，真正修改时才可能复制
- 多线程同时读写同一个可变集合不安全，需要 actor、锁、串行队列或不可变快照

---

## 4. 编译流程

### 4.1 从源码到机器码

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

### 4.2 SIL 的作用

SIL 是 Swift 编译器特有的中间表示，保留了 Swift 的高级语义，便于优化：

- ARC 插入与消除
- 闭包捕获分析
- 泛型特化
- 函数内联
- 去虚拟化
- 所有权检查
- async/await 状态机转换

### 4.3 编译优化示例

```swift
final class Calculator {
    func add(_ a: Int, _ b: Int) -> Int {
        a + b
    }
}
```

`final` 表示不能被继承或重写，编译器可以把动态派发优化为直接调用，甚至进一步内联。

---

## 5. 类型系统与内存布局

### 5.1 值类型和引用类型

| 类型 | 示例 | 语义 | 常见存储 |
| --- | --- | --- | --- |
| 值类型 | `struct`、`enum`、`tuple` | 赋值产生独立值 | 栈、堆、寄存器均可能 |
| 引用类型 | `class` | 赋值共享对象引用 | 堆 |

值类型不等于一定在栈上。它可能因为闭包捕获、泛型容器、存在类型装箱、类属性等原因存储在堆上。

### 5.2 Struct 内存布局

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

### 5.3 Class 内存布局

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

### 5.4 Enum 内存布局

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

## 6. 方法调度

### 6.1 常见调度方式

| 调度方式 | 触发场景 | 性能 | 动态性 |
| --- | --- | --- | --- |
| 直接调用 | `struct`、`enum`、`final`、`private` | 最高 | 最低 |
| vtable | 普通 Swift class 方法 | 高 | 支持重写 |
| witness table | 协议泛型调用 | 中 | 支持协议抽象 |
| ObjC 消息发送 | `@objc`、`dynamic`、NSObject 子类 | 较低 | 最高 |

### 6.2 直接调用

```swift
struct Adder {
    func add(_ a: Int, _ b: Int) -> Int {
        a + b
    }
}
```

结构体方法静态可知，编译器可以直接调用、内联和优化。

### 6.3 vtable 调用

```swift
class Animal {
    func speak() {}
}

class Dog: Animal {
    override func speak() {}
}
```

类支持继承和重写，方法地址通过虚函数表查找。

### 6.4 协议见证表

```swift
protocol Drawable {
    func draw()
}

func render<T: Drawable>(_ value: T) {
    value.draw()
}
```

编译器通过 Protocol Witness Table 找到具体类型对协议方法的实现。

### 6.5 `@objc` 与 `dynamic`

```swift
@objc dynamic func reload() {}
```

- `@objc`：暴露给 Objective-C Runtime
- `dynamic`：强制动态派发，禁止编译器静态化
- `@objcMembers`：让类成员默认暴露给 ObjC，需谨慎使用

不需要 ObjC 动态能力时，应避免过度使用。

---

## 7. 泛型与面向协议编程

### 7.1 泛型解决什么问题

泛型用于编写“与具体类型无关，但保留类型安全”的代码。它避免为 `Int`、`String`、自定义 Model 分别写重复逻辑。

```swift
func swapValues<T>(_ a: inout T, _ b: inout T) {
    let temp = a
    a = b
    b = temp
}

var left = 1
var right = 2
swapValues(&left, &right)

var first = "A"
var second = "B"
swapValues(&first, &second)
```

`T` 是类型参数，表示调用时由编译器推断出的具体类型。上面的函数既能处理 `Int`，也能处理 `String`，但不会把 `Int` 和 `String` 混用。

### 7.2 泛型函数和类型约束

泛型通常需要配合约束，否则函数内部不知道这个类型具备什么能力。

```swift
func maxValue<T: Comparable>(_ a: T, _ b: T) -> T {
    a > b ? a : b
}
```

`T: Comparable` 表示 `T` 必须遵守 `Comparable` 协议，因此函数内部可以使用 `>` 比较。

多个约束可以写在 `where` 后面：

```swift
func allMatch<C1: Collection, C2: Collection>(
    _ first: C1,
    _ second: C2
) -> Bool where C1.Element == C2.Element, C1.Element: Equatable {
    first.count == second.count && zip(first, second).allSatisfy { pair in
        pair.0 == pair.1
    }
}
```

这个例子表达了两个条件：

- 两个集合的元素类型必须相同
- 元素必须遵守 `Equatable`，才能比较是否相等

### 7.3 泛型类型

泛型也可以用于 `struct`、`class`、`enum`。

```swift
struct Stack<Element> {
    private var values: [Element] = []

    var isEmpty: Bool {
        values.isEmpty
    }

    mutating func push(_ value: Element) {
        values.append(value)
    }

    mutating func pop() -> Element? {
        values.popLast()
    }
}

var intStack = Stack<Int>()
intStack.push(1)
intStack.push(2)
let value = intStack.pop()
```

`Array<Element>`、`Dictionary<Key, Value>`、`Set<Element>` 本质上都是标准库提供的泛型集合。

### 7.4 泛型和协议关联类型

协议可以通过 `associatedtype` 声明关联类型。`associatedtype` 表示“遵守协议的具体类型自己决定这里是什么类型”。

```swift
protocol Repository {
    associatedtype Entity

    func fetchAll() async throws -> [Entity]
    func save(_ entity: Entity) async throws
}

struct UserRepository: Repository {
    func fetchAll() async throws -> [User] {
        []
    }

    func save(_ entity: User) async throws {
        // 保存用户
    }
}
```

`UserRepository` 实现协议时，编译器会推断 `Entity == User`。

使用带有关联类型的协议时，常见方式是继续使用泛型：

```swift
final class ListViewModel<R: Repository> {
    private let repository: R
    private(set) var items: [R.Entity] = []

    init(repository: R) {
        self.repository = repository
    }

    func load() async throws {
        items = try await repository.fetchAll()
    }
}
```

这样可以保留 `R.Entity` 的具体类型信息，避免过早擦除类型。

### 7.5 面向协议编程

面向协议编程通常称为 POP，也就是 Protocol-Oriented Programming。它的核心不是“到处写 protocol”，而是用协议表达能力和边界，用值类型、泛型和扩展组合行为。

面向协议编程常用于：

- 定义模块边界，隐藏具体实现
- 让业务依赖抽象，便于测试
- 给多个类型复用默认实现
- 用小协议组合出更大的能力
- 避免为了复用逻辑强行继承 class

示例：用协议隔离网络层，便于替换真实实现和测试实现。

```swift
protocol UserServicing {
    func fetchUser(id: Int) async throws -> User
}

struct RemoteUserService: UserServicing {
    func fetchUser(id: Int) async throws -> User {
        // 真实网络请求
        User(id: id, name: "Remote")
    }
}

struct MockUserService: UserServicing {
    func fetchUser(id: Int) async throws -> User {
        // 测试或预览数据
        User(id: id, name: "Mock")
    }
}

@MainActor
final class UserViewModel<Service: UserServicing>: ObservableObject {
    @Published private(set) var name = ""

    private let service: Service

    init(service: Service) {
        self.service = service
    }

    func load(id: Int) async throws {
        let user = try await service.fetchUser(id: id)
        name = user.name
    }
}
```

这里 `UserViewModel` 不关心服务来自网络、缓存还是测试数据，只关心它具备 `fetchUser` 能力。

### 7.6 协议扩展和默认实现

协议扩展可以为所有遵守协议的类型提供默认行为。

```swift
protocol ReusableView {
    static var reuseIdentifier: String { get }
}

extension ReusableView {
    static var reuseIdentifier: String {
        String(describing: Self.self)
    }
}

final class UserCell: UITableViewCell, ReusableView {}

let identifier = UserCell.reuseIdentifier
```

这个例子避免每个 cell 手写重复的复用标识。

也可以通过约束让默认实现只对部分类型生效：

```swift
protocol JSONDecoding {
    associatedtype Model: Decodable
    var decoder: JSONDecoder { get }
}

extension JSONDecoding {
    func decode(_ type: Model.Type, from data: Data) throws -> Model {
        try decoder.decode(type, from: data)
    }
}
```

### 7.7 协议组合

小协议可以组合使用，避免一个大协议要求实现过多无关能力。

```swift
protocol IdentifiableModel {
    var id: Int { get }
}

protocol Displayable {
    var title: String { get }
}

func renderItem<T: IdentifiableModel & Displayable>(_ item: T) {
    print("\(item.id): \(item.title)")
}
```

设计建议：

- 协议尽量表达稳定能力，而不是临时参数集合
- 协议方法数量不要过多，否则容易变成“胖接口”
- 如果只有一个实现且短期不会替换，不必强行抽协议
- 面向协议不等于排斥 class，生命周期和共享状态明确时 class 仍然合适

### 7.8 `some Protocol`

`some` 是不透明类型。调用方不知道具体类型，但编译器知道。它适合隐藏返回值的具体类型，同时保留静态类型和优化空间。

```swift
func makeTitle() -> some View {
    Text("Hello")
        .font(.headline)
}
```

要求函数所有返回路径的底层具体类型一致：

```swift
func makeLabel(isImportant: Bool) -> some View {
    if isImportant {
        return Text("Important").bold()
    } else {
        return Text("Normal")
    }
}
```

这个示例在 SwiftUI 中可能因为两个分支底层类型不同而无法通过。实际可以使用 `@ViewBuilder` 或统一返回类型解决。

### 7.9 `any Protocol`

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

- 算法和热路径优先泛型，类型信息完整，优化空间更好
- API 隐藏具体返回类型用 `some`，调用方不需要知道具体类型
- 异构集合、插件化、运行时动态选择实现时用 `any`
- 带有关联类型或 `Self` 要求的协议，优先考虑泛型，确实需要动态存储时再做类型擦除

### 7.10 类型擦除

类型擦除用于把不同具体类型包装成同一种对外类型。标准库里的 `AnySequence`、Combine 里的 `AnyPublisher` 都是典型例子。

简化示例：

```swift
protocol EventTracking {
    func track(name: String)
}

struct ConsoleTracker: EventTracking {
    func track(name: String) {
        print(name)
    }
}

struct AnyEventTracker: EventTracking {
    private let trackClosure: (String) -> Void

    init<T: EventTracking>(_ tracker: T) {
        self.trackClosure = tracker.track
    }

    func track(name: String) {
        trackClosure(name)
    }
}

let tracker: AnyEventTracker = AnyEventTracker(ConsoleTracker())
```

类型擦除适合把复杂泛型藏在模块边界后面，但会引入额外间接调用，热路径要谨慎。

### 7.11 泛型底层实现

Swift 泛型常通过类型元数据和 witness table 支持。对性能敏感路径，编译器可能进行泛型特化，生成具体类型版本。

```swift
func containsValue<T: Equatable>(_ values: [T], target: T) -> Bool {
    values.contains(target)
}
```

当调用 `containsValue([1, 2, 3], target: 2)` 时，编译器可能生成 `Int` 专用版本，减少抽象成本。

---

## 8. 函数式编程

函数式编程强调把计算表达为值转换，减少共享可变状态和副作用。Swift 不是纯函数式语言，但标准库提供了很多函数式工具，例如 `map`、`filter`、`reduce`、`compactMap`、`flatMap`、`forEach`。

### 8.1 纯函数和副作用

纯函数的特点：

- 相同输入总是得到相同输出
- 不修改外部状态
- 不依赖时间、随机数、网络、磁盘等外部环境

```swift
func discountedPrice(price: Double, rate: Double) -> Double {
    price * (1 - rate)
}
```

有副作用的函数：

```swift
var total = 0

func addToTotal(_ value: Int) {
    total += value // 修改外部状态，是副作用
}
```

工程中不是不能有副作用，而是要把副作用集中在边界，例如网络、数据库、日志、UI 更新；核心计算尽量写成纯函数，便于测试和复用。

### 8.2 map、filter、reduce

```swift
struct Order {
    let id: Int
    let amount: Double
    let isPaid: Bool
}

let orders = [
    Order(id: 1, amount: 20, isPaid: true),
    Order(id: 2, amount: 35, isPaid: false),
    Order(id: 3, amount: 50, isPaid: true)
]

let paidAmounts = orders
    .filter { $0.isPaid }     // 保留已支付订单
    .map { $0.amount }        // 提取金额

let totalPaid = paidAmounts
    .reduce(0, +)             // 求和
```

可读性建议：链式调用超过三四步，或闭包逻辑变复杂时，拆出中间变量或命名函数。

### 8.3 compactMap 和 flatMap

`compactMap` 用于转换并丢弃 `nil`。

```swift
let inputs = ["1", "2", "abc", "4"]
let numbers = inputs.compactMap { Int($0) } // [1, 2, 4]
```

`flatMap` 常用于把嵌套集合摊平。

```swift
let sections = [
    ["A", "B"],
    ["C"],
    ["D", "E"]
]

let items = sections.flatMap { $0 } // ["A", "B", "C", "D", "E"]
```

### 8.4 forEach 和 for-in

`forEach` 适合简单遍历执行动作，但不能使用 `break`、`continue` 提前控制循环。

```swift
users.forEach { user in
    print(user.name)
}
```

需要提前退出时用 `for-in`：

```swift
for user in users {
    if user.id == targetID {
        print(user)
        break
    }
}
```

### 8.5 高阶函数

高阶函数是接收函数作为参数，或返回函数的函数。

```swift
func makeValidator(minLength: Int) -> (String) -> Bool {
    { text in
        text.count >= minLength
    }
}

let passwordValidator = makeValidator(minLength: 8)
let isValid = passwordValidator("12345678")
```

在 iOS 中，高阶函数常用于：

- 表单校验规则
- 数据排序和筛选条件
- 回调处理
- 中间件、拦截器、埋点包装

### 8.6 函数式处理业务数据

示例：把接口返回的商品列表转换成 UI Model。

```swift
struct ProductDTO {
    let id: Int
    let name: String?
    let price: Double
    let isOnSale: Bool
}

struct ProductCellModel {
    let id: Int
    let title: String
    let priceText: String
}

let cellModels = products
    .filter { $0.isOnSale }
    .compactMap { product -> ProductCellModel? in
        guard let name = product.name, !name.isEmpty else {
            return nil
        }

        return ProductCellModel(
            id: product.id,
            title: name,
            priceText: String(format: "%.2f", product.price)
        )
    }
    .sorted { $0.title < $1.title }
```

这类写法适合“输入集合 -> 过滤 -> 转换 -> 排序 -> 输出集合”的数据管道。

### 8.7 函数式编程边界

| 场景 | 建议 |
| --- | --- |
| 简单集合转换 | 使用 `map`、`filter`、`compactMap` |
| 聚合统计 | 使用 `reduce`，但逻辑复杂时用普通循环 |
| 需要提前退出 | 使用 `for-in` |
| 需要处理错误 | 使用 `do/catch`、`Result` 或 `throws`，不要硬塞进链式调用 |
| 大数据热路径 | 注意中间数组分配，必要时用 `lazy` 或单次循环 |
| UI 更新、网络、数据库 | 允许副作用，但放在明确边界 |

`lazy` 可以减少中间集合创建：

```swift
let firstLargeEven = numbers
    .lazy
    .filter { $0.isMultiple(of: 2) }
    .first { $0 > 1_000 }
```

---

## 9. Copy-on-Write

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

## 10. ARC 与内存管理

### 10.1 ARC 工作方式

Swift 类实例通过 ARC 管理生命周期。编译器在合适位置插入 retain/release，并在 SIL 阶段优化冗余引用计数操作。

### 10.2 强引用循环

```swift
final class Owner {
    var pet: Pet?
}

final class Pet {
    var owner: Owner?
}
```

双方强引用会导致无法释放。可用 `weak` 或 `unowned` 打破。

### 10.3 `weak` 与 `unowned`

| 修饰符 | 是否可选 | 对象释放后 | 场景 |
| --- | --- | --- | --- |
| `weak` | 必须可选 | 自动变 nil | 生命周期可能更短 |
| `unowned` | 非可选 | 访问崩溃 | 被引用对象生命周期更长 |

```swift
final class Child {
    weak var parent: Parent?
}
```

### 10.4 闭包捕获

```swift
service.load { [weak self] result in
    guard let self else { return }
    self.render(result)
}
```

不要机械地给所有闭包加 `weak`。非逃逸闭包或立即执行闭包通常不需要。

---

## 11. Swift 与 Objective-C 互操作

### 11.1 Swift 调 Objective-C

通过 Bridging Header 暴露 Objective-C 头文件：

```objc
// Project-Bridging-Header.h
#import "UserService.h"
```

Swift 中可直接使用：

```swift
let service = UserService()
```

### 11.2 Objective-C 调 Swift

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

### 11.3 桥接成本

常见桥接：

- `String` ↔ `NSString`
- `Array` ↔ `NSArray`
- `Dictionary` ↔ `NSDictionary`
- Swift closure ↔ ObjC Block

在高频路径中大量桥接可能产生性能成本。

---

## 12. Swift 多线程与 Concurrency

Swift 中“多线程”和“并发”不是同一个概念：

- 多线程关注代码实际运行在哪些线程上
- 并发关注多个任务在同一时间段内推进
- 异步关注任务等待时不阻塞当前执行流
- 并行关注多个任务真正同时在不同 CPU core 上执行

现代 Swift 优先使用 Swift Concurrency，也就是 `async/await`、`Task`、`TaskGroup`、`actor`、`MainActor`、`Sendable` 等语言级并发能力。存量 iOS 工程仍会大量遇到 GCD、OperationQueue、Thread、锁和 RunLoop。

### 12.1 主线程和 UI 更新

iOS 中 UIKit 和 SwiftUI 的 UI 状态更新必须回到主线程。Swift Concurrency 中推荐用 `@MainActor` 表达“这个类型或方法在主 actor 上执行”。

```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    @Published private(set) var name = ""

    private let service: ProfileService

    init(service: ProfileService) {
        self.service = service
    }

    func load() async {
        // 网络请求在挂起期间不会阻塞主线程。
        let profile = await service.fetchProfile()

        // 当前类型被 @MainActor 隔离，这里可以安全更新 UI 状态。
        name = profile.name
    }
}
```

注意：`@MainActor` 保证隔离，不代表耗时计算会自动离开主线程。CPU 密集任务应放到后台任务或专门服务中处理。

### 12.2 async/await

`async/await` 不等于创建线程。它把异步逻辑写成顺序代码，底层通过任务挂起和恢复实现。

```swift
func load() async throws -> User {
    let data = try await api.fetch()
    return try decoder.decode(User.self, from: data)
}
```

编译器会把异步函数转换为状态机。

错误处理和取消示例：

```swift
func loadUser() async {
    do {
        let user = try await api.fetchUser()
        print(user)
    } catch is CancellationError {
        // 任务被取消，通常不展示错误 toast。
        print("cancelled")
    } catch {
        // 普通错误，例如网络失败、解析失败。
        print(error.localizedDescription)
    }
}
```

### 12.3 Task 与结构化并发

`Task` 表示一个并发任务。结构化并发强调父子任务关系，父任务结束、抛错或取消时，子任务也应被管理。

```swift
async let profile = api.fetchProfile()
async let settings = api.fetchSettings()

let result = try await (profile, settings)
```

`async let` 适合数量固定、生命周期清晰的并发请求。

```swift
func loadHome() async throws -> HomeData {
    async let banner = api.fetchBanner()
    async let feed = api.fetchFeed()

    // 两个请求并发启动，等待时按结构化关系统一收敛。
    let (bannerValue, feedValue) = try await (banner, feed)
    return HomeData(banner: bannerValue, feed: feedValue)
}
```

动态数量任务使用 `TaskGroup`：

```swift
func fetchImages(urls: [URL]) async throws -> [Data] {
    try await withThrowingTaskGroup(of: Data.self) { group in
        for url in urls {
            group.addTask {
                try await URLSession.shared.data(from: url).0
            }
        }

        var results: [Data] = []
        for try await data in group {
            results.append(data)
        }
        return results
    }
}
```

创建非结构化任务时要小心生命周期：

```swift
let task = Task {
    try await api.refresh()
}

task.cancel() // 需要明确取消，否则任务可能继续运行
```

### 12.4 Actor

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

适合 actor 的场景：

- 缓存读写
- Token 刷新合并
- 用户会话状态
- 跨任务共享的可变字典、数组、计数器

### 12.5 Actor 重入

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

### 12.6 Sendable

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

`@unchecked Sendable` 的含义是“编译器不检查，我保证线程安全”。它不是修复警告的万能开关，必须配合锁、串行队列、不可变状态或其他同步手段。

### 12.7 GCD

GCD 是 Grand Central Dispatch，通过队列提交任务，由系统线程池调度执行。常见队列：

| 队列 | 说明 | 场景 |
| --- | --- | --- |
| `DispatchQueue.main` | 主队列，运行在主线程 | UI 更新 |
| global queue | 全局并发队列 | 后台计算、I/O、网络回调处理 |
| serial queue | 自定义串行队列 | 保护共享状态、顺序执行 |
| concurrent queue | 自定义并发队列 | 多读单写、批量任务 |

```swift
DispatchQueue.global(qos: .userInitiated).async {
    // 后台执行耗时任务
    let image = decodeImage()

    DispatchQueue.main.async {
        // 回主线程更新 UI
        imageView.image = image
    }
}
```

QoS 是 Quality of Service，用来告诉系统任务重要程度：

| QoS | 场景 |
| --- | --- |
| `.userInteractive` | 动画、触摸响应，必须立即完成 |
| `.userInitiated` | 用户主动触发且等待结果 |
| `.utility` | 下载、导入、批处理，可等待 |
| `.background` | 清理、预取、低优先级维护任务 |

串行队列保护共享状态：

```swift
final class LegacyCache {
    private let queue = DispatchQueue(label: "cache.serial.queue")
    private var storage: [String: Data] = [:]

    func set(_ data: Data, for key: String) {
        queue.async {
            self.storage[key] = data
        }
    }

    func data(for key: String) -> Data? {
        queue.sync {
            storage[key]
        }
    }
}
```

注意不要在同一个串行队列内部再次 `sync` 到自己，否则会死锁。

### 12.8 OperationQueue

`OperationQueue` 比 GCD 更面向对象，支持依赖、取消、最大并发数。

```swift
let download = BlockOperation {
    print("download")
}

let parse = BlockOperation {
    print("parse")
}

parse.addDependency(download) // parse 必须等 download 完成

let queue = OperationQueue()
queue.maxConcurrentOperationCount = 2
queue.addOperations([download, parse], waitUntilFinished: false)
```

适合场景：

- 多步骤任务有依赖关系
- 需要控制最大并发数
- 需要取消队列中还没执行的任务
- 老项目已经基于 Operation 封装下载、上传、图片处理管线

### 12.9 Thread 和 RunLoop

`Thread` 是更底层的线程 API，现代业务代码很少直接使用。它适合需要固定线程和 RunLoop 的场景，例如长期 Socket、音频、特定 C API 线程亲和。

```swift
let thread = Thread {
    autoreleasepool {
        let runLoop = RunLoop.current
        runLoop.add(Port(), forMode: .default)

        while !Thread.current.isCancelled {
            runLoop.run(mode: .default, before: Date(timeIntervalSinceNow: 0.1))
        }
    }
}

thread.name = "com.example.worker"
thread.start()
```

普通异步任务不要直接创建 Thread，优先选择 Swift Concurrency、GCD 或 OperationQueue。

### 12.10 锁和线程安全

当多个线程或任务同时访问同一份可变状态时，需要同步。常见方式：

| 方式 | 特点 | 适合场景 |
| --- | --- | --- |
| `actor` | 语言级隔离，适配 Swift Concurrency | 新代码、异步共享状态 |
| `NSLock` | 互斥锁，简单直接 | 短临界区同步 |
| `os_unfair_lock` | 低层锁，性能高但使用要求高 | 性能敏感基础设施 |
| 串行 `DispatchQueue` | 顺序执行，容易理解 | 存量代码共享状态 |
| 不可变快照 | 避免共享可变状态 | 读多写少、UI 状态传递 |

`NSLock` 示例：

```swift
final class LockedCounter: @unchecked Sendable {
    private let lock = NSLock()
    private var value = 0

    func increment() {
        lock.lock()
        defer { lock.unlock() }
        value += 1
    }

    func current() -> Int {
        lock.lock()
        defer { lock.unlock() }
        return value
    }
}
```

锁使用原则：

- 临界区尽量短
- 不要在持锁期间执行网络请求、磁盘 I/O 或回调外部代码
- 固定加锁顺序，避免死锁
- 新 Swift 并发代码优先考虑 actor

### 12.11 常见多线程问题

| 问题 | 原因 | 解决方向 |
| --- | --- | --- |
| UI 偶发崩溃 | 子线程更新 UIKit/SwiftUI 状态 | `@MainActor` 或回主队列 |
| 数据竞争 | 多线程同时读写变量或集合 | actor、锁、串行队列 |
| 死锁 | 当前串行队列中同步等待自己 | 避免嵌套 `sync` |
| 优先级反转 | 高优先级任务等待低优先级任务持有的锁 | 缩短锁、合理 QoS |
| 任务泄漏 | 非结构化 Task 未取消 | 保存 Task 并在生命周期结束时取消 |
| 重复请求 | 多个任务同时刷新 Token | actor 内保存进行中的 Task |
| 主线程卡顿 | CPU、I/O、解码、JSON 解析在主线程 | 移到后台，结果回主线程 |

### 12.12 选择建议

| 需求 | 推荐 |
| --- | --- |
| 新代码异步接口 | `async/await` |
| 并发请求数量固定 | `async let` |
| 并发请求数量动态 | `TaskGroup` |
| 共享可变状态 | `actor` |
| UI 状态更新 | `@MainActor` |
| 存量回调代码调度线程 | GCD |
| 有依赖和取消的任务管线 | OperationQueue |
| 固定线程和 RunLoop | Thread |
| 极短同步临界区 | `NSLock` 或系统锁 |

---

## 13. 性能优化要点

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

## 14. 面试高频问题

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

7. **Swift 里有哪些集合类型？**
   最常用的是 `Array`、`Dictionary`、`Set`。`Array` 有序可重复，`Dictionary` 按 key 存取，`Set` 无序不重复并支持集合运算。

8. **Array、Dictionary、Set 是线程安全的吗？**
   单独值本身不是共享可变状态时没有问题；多个线程同时读写同一个可变集合不安全，需要 actor、锁、串行队列或不可变快照。

9. **GCD 和 Swift Concurrency 怎么选？**
   新代码优先 Swift Concurrency；存量回调、底层调度和简单切线程仍常用 GCD。两者可以共存，但不要混到生命周期和取消关系不清楚。

10. **什么是面向协议编程？**
   用协议表达能力和边界，用泛型、扩展和默认实现组合行为。它适合模块解耦、测试替身和复用能力，但不等于所有地方都要抽协议。

11. **函数式编程在 Swift 中常见在哪里？**
   常见于集合转换、数据清洗、表单校验、排序筛选和 UI Model 构建。简单数据管道适合链式写法，复杂逻辑或需要提前退出时普通循环更清晰。

---

## 15. 总结

Swift 的重点包括基础语法、值语义、集合、泛型、面向协议编程、函数式编程、类型系统、编译优化、方法派发、ARC、COW 和并发模型。相比 Objective-C，Swift 更倾向于静态类型和编译期优化，但在与 Objective-C 互操作、协议抽象和并发场景中仍有运行时成本。

写高质量 Swift 代码的关键是：默认使用值语义和静态派发，需要动态性时再显式引入 `class`、`any`、`@objc`、`dynamic` 等能力。
