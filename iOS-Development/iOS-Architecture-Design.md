# iOS 架构设计

## 1. 架构设计的目标

架构不是目录分层，也不是模式堆叠。架构的核心目标是让代码在业务增长后仍然可理解、可修改、可测试、可替换。

好的架构应该解决：

- 页面逻辑过重
- 网络、缓存、数据库混杂
- 跳转流程难维护
- 状态来源混乱
- 单元测试困难
- 模块间依赖失控
- 新需求改动范围不可控

架构设计的基本原则：

| 原则 | 含义 |
| --- | --- |
| 单一职责 | 一个类型只承担清晰职责 |
| 关注点分离 | UI、业务、数据、导航分离 |
| 依赖倒置 | 高层模块依赖抽象，不依赖细节 |
| 可测试性 | 核心逻辑可脱离 UI 测试 |
| 渐进演进 | 根据复杂度增加层次，不提前过度设计 |

---

## 2. MVC

### 2.1 经典 MVC

```text
Model
  ↑ ↓
Controller
  ↑ ↓
View
```

职责：

- Model：数据和业务规则
- View：展示和用户输入
- Controller：协调 Model 和 View

### 2.2 Cocoa Touch MVC

iOS 中常见形态：

```text
ViewController
  ├── 持有 View
  ├── 请求数据
  ├── 处理回调
  ├── 更新 UI
  ├── 处理导航
  └── 处理业务分支
```

问题是容易形成 Massive View Controller。

### 2.3 MVC 适用场景

适合：

- 简单页面
- 原型开发
- 展示逻辑很少的页面
- 小型项目

不适合：

- 复杂表单
- 多状态页面
- 强测试要求
- 多步骤流程
- 复杂业务逻辑

### 2.4 MVC 优化

即使使用 MVC，也可以拆分：

- Network Service
- Data Mapper
- TableView DataSource 对象
- Form Validator
- Router/Coordinator
- ViewModel-like Formatter

---

## 3. MVVM

### 3.1 MVVM 核心思想

```text
View / ViewController
  ↓ 用户输入
ViewModel
  ↓ 请求数据
Model / Service
  ↑ 返回结果
ViewModel
  ↓ 输出状态
View
```

ViewModel 负责：

- 页面状态
- 输入处理
- 数据转换
- 表单校验
- 请求编排
- 错误文案转换

ViewController 负责：

- 创建视图
- 绑定状态
- 转发用户事件
- 执行动画和导航触发

### 3.2 ViewModel 示例

```swift
protocol UserService {
    func fetchUser(id: String) async throws -> User
}

@MainActor
final class ProfileViewModel: ObservableObject {
    @Published private(set) var state: State = .idle

    private let userID: String
    private let service: UserService

    init(userID: String, service: UserService) {
        self.userID = userID
        self.service = service
    }

    func load() async {
        state = .loading
        do {
            let user = try await service.fetchUser(id: userID)
            state = .loaded(user)
        } catch {
            state = .failed(error.localizedDescription)
        }
    }

    enum State {
        case idle
        case loading
        case loaded(User)
        case failed(String)
    }
}
```

### 3.3 MVVM 常见误区

- ViewModel 不是 Model 的简单包装
- ViewModel 不应直接持有具体 View
- ViewModel 不应承担导航细节
- 不要给每个小控件都创建 ViewModel
- 不要把所有业务都塞进 ViewModel，复杂业务应下沉到 UseCase

---

## 4. Coordinator / Router

### 4.1 为什么需要 Coordinator

页面跳转散落在 ViewController 中会导致：

- 页面之间强耦合
- 流程复用困难
- 测试困难
- 深链和多入口处理复杂

Coordinator 负责流程编排：

```text
ViewController
  ↓ 用户事件
Coordinator
  ↓ 创建页面并跳转
Next ViewController
```

### 4.2 Coordinator 示例

```swift
final class ProfileCoordinator {
    private let navigationController: UINavigationController
    private let service: UserService

    init(navigationController: UINavigationController, service: UserService) {
        self.navigationController = navigationController
        self.service = service
    }

    func showProfile(userID: String) {
        let viewModel = ProfileViewModel(userID: userID, service: service)
        let viewController = ProfileViewController(viewModel: viewModel)
        navigationController.pushViewController(viewController, animated: true)
    }
}
```

Coordinator 应该做：

- 页面创建
- 依赖注入
- push/present/dismiss
- 多步骤流程控制

不应该做：

- 发网络请求
- 写业务规则
- 持有大量页面状态
- 处理表单校验

---

## 5. Clean Architecture 分层

常见分层：

```text
Presentation
  ├── View
  └── ViewModel

Domain
  ├── Entity
  └── UseCase

Data
  ├── Repository
  ├── RemoteDataSource
  └── LocalDataSource
```

依赖方向：

```text
Presentation -> Domain <- Data
```

Domain 不依赖 UIKit、网络库、数据库框架。

### 5.1 UseCase

UseCase 表示一个业务动作：

```swift
final class LoadProfileUseCase {
    private let repository: UserRepository

    init(repository: UserRepository) {
        self.repository = repository
    }

    func execute(userID: String) async throws -> Profile {
        let user = try await repository.user(id: userID)
        return Profile(user: user, canEdit: user.isCurrentUser)
    }
}
```

### 5.2 Repository

Repository 管理数据来源：

```swift
final class UserRepository {
    private let remote: UserRemoteDataSource
    private let cache: UserCache

    func user(id: String) async throws -> User {
        if let cached = cache.value(for: id) {
            return cached
        }

        let user = try await remote.fetchUser(id: id)
        cache.save(user)
        return user
    }
}
```

---

## 6. 依赖注入

依赖注入让对象不自己创建依赖，而是从外部传入。

```swift
final class LoginViewModel {
    private let authService: AuthService

    init(authService: AuthService) {
        self.authService = authService
    }
}
```

好处：

- 可替换实现
- 易于测试
- 降低耦合
- 依赖关系明确

测试替身：

```swift
struct MockAuthService: AuthService {
    func login(account: String, password: String) async throws -> Token {
        Token(value: "mock")
    }
}
```

实践建议：

- 小项目用初始化注入
- 中大型项目可用 Assembly 或轻量容器
- 不要滥用全局单例
- 协议只抽象稳定边界，不要为每个类强行建协议

---

## 7. 响应式架构

### 7.1 Combine / RxSwift 适用场景

适合：

- 表单校验
- 搜索防抖
- 多接口组合
- 状态绑定
- 事件流处理

示例：

```swift
searchTextPublisher
    .debounce(for: .milliseconds(300), scheduler: RunLoop.main)
    .removeDuplicates()
    .flatMap { query in
        service.search(query)
    }
    .receive(on: RunLoop.main)
    .sink { results in
        self.results = results
    }
```

### 7.2 响应式误区

- 不要为了响应式而响应式
- 简单回调不一定要包装成流
- 注意订阅生命周期
- 注意错误流终止
- 避免链路过长导致调试困难

---

## 8. SwiftUI 架构

### 8.1 状态归属

| 属性包装器 | 状态归属 | 场景 |
| --- | --- | --- |
| `@State` | 当前 View 私有 | 局部 UI 状态 |
| `@Binding` | 父级传入 | 子 View 修改父状态 |
| `@StateObject` | 当前 View 创建并持有 | 页面 ViewModel |
| `@ObservedObject` | 外部传入 | 父级管理对象 |
| `@EnvironmentObject` | 环境共享 | 跨层级共享状态 |
| `@Environment` | 环境值 | 主题、尺寸、上下文 |

原则：

- 状态放在最小公共父级
- View 不直接依赖网络层
- ViewModel 不依赖具体 View
- 全局状态要少且边界清晰

### 8.2 SwiftUI MVVM

```swift
struct ProfileView: View {
    @StateObject private var viewModel: ProfileViewModel

    var body: some View {
        content
            .task {
                await viewModel.load()
            }
    }
}
```

### 8.3 TCA 思路

TCA 核心元素：

- State
- Action
- Reducer
- Effect
- Store

适合大型 SwiftUI 项目，优势是单向数据流、可测试、状态清晰；成本是样板代码较多，团队需要统一理解。

---

## 9. 模块化设计

### 9.1 常见模块划分

```text
App
├── FeatureHome
├── FeatureProfile
├── FeaturePayment
├── CoreNetwork
├── CoreDatabase
├── CoreUI
├── CoreAnalytics
└── Domain
```

依赖方向应清晰：

```text
Feature -> Domain -> Core Abstractions
Feature -> CoreUI
Feature -> CoreNetwork Implementation
```

### 9.2 模块化收益

- 降低编译影响范围
- 明确边界
- 支持多人并行
- 便于复用
- 便于拆分测试

### 9.3 模块化风险

- 循环依赖
- Core 模块膨胀
- 过早拆分
- 跨模块路由复杂
- 资源和多语言管理复杂

---

## 10. 架构选择

| 项目情况 | 推荐 |
| --- | --- |
| 小型展示项目 | MVC |
| 中等复杂页面 | MVVM |
| 复杂导航流程 | MVVM + Coordinator |
| 复杂业务规则 | MVVM + UseCase + Repository |
| 大型 SwiftUI | TCA 或单向数据流 |
| 遗留 UIKit | 渐进拆分，新增页面采用 MVVM-C |

决策原则：

- 先识别复杂度和变化点
- 不为简单页面强行分层
- 对核心业务做可测试设计
- 对跨页面流程引入 Coordinator
- 对数据来源复杂场景引入 Repository

---

## 11. 架构腐化信号

常见信号：

- ViewController 超过数千行
- 网络请求散落在 UI 层
- 单例到处被直接调用
- 页面跳转互相强依赖
- ViewModel 持有 View
- Repository 写业务 UI 文案
- Core 模块被所有业务随意依赖
- 单元测试无法构造对象

治理手段：

- 明确依赖方向
- 建立模块边界
- 引入依赖注入
- 把业务规则下沉到 UseCase
- 把导航下沉到 Coordinator
- 为核心逻辑补单元测试

---

## 12. 面试高频问题

1. **MVC 和 MVVM 的区别？**  
   MVVM 把展示状态和业务转换逻辑从 ViewController 抽到 ViewModel，降低 ViewController 复杂度并提升可测试性。

2. **ViewModel 能不能 import UIKit？**  
   最好不要。ViewModel 应尽量与 UI 框架解耦，但展示模型中少量 UI 适配可以通过边界层处理。

3. **Coordinator 解决什么问题？**  
   解决导航和流程编排散落在 ViewController 中的问题。

4. **Repository 和 UseCase 区别？**  
   Repository 管数据来源和缓存策略；UseCase 表达业务动作和业务规则。

5. **什么时候需要模块化？**  
   当业务边界清晰、团队并行、编译耗时或复用需求明显时；过早模块化会增加成本。

6. **架构设计最重要的原则？**  
   依赖方向清晰，复杂度与收益匹配，核心逻辑可测试。

---

## 13. 总结

iOS 架构设计应从业务复杂度出发，而不是从模式出发。简单页面可以保持简单；复杂页面用 MVVM 管状态；复杂流程用 Coordinator 管导航；复杂业务用 UseCase 和 Repository 分离规则与数据来源；大型项目再引入模块化和依赖治理。

架构的最终价值是降低长期维护成本，而不是增加形式感。

---

## 14. 架构落地与评审清单

### 14.1 从需求到模块

架构应先识别业务变化点，再决定抽象，而不是从目录或设计模式开始。

```text
业务能力与变化点
      ↓
领域边界和模块职责
      ↓
模块公开接口
      ↓
数据流、依赖方向、错误模型
      ↓
具体框架和实现
```

划分模块时重点判断：

- 哪些代码因同一业务原因变化
- 哪些能力需要独立测试、替换或发布
- 数据由谁创建、修改和销毁
- 哪些跨模块调用属于稳定契约
- 哪些平台或第三方实现应被隔离

### 14.2 依赖规则

```text
App / Composition Root
          ↓
Feature / Presentation
          ↓
Domain
          ↑
Data / Platform Implementations
```

核心规则：

- Domain 不依赖 UI、数据库和网络框架
- Feature 不访问其他 Feature 的内部实现
- 具体实现通过协议注入
- App 层组装依赖，不承载业务规则
- 跨模块模型最小化，避免共享巨型 Model

### 14.3 数据流与状态

复杂页面优先采用单向数据流：

```text
User Action
    ↓
Intent / Event
    ↓
Reducer / ViewModel / UseCase
    ↓
New State
    ↓
View
```

状态设计应满足：

- 单一可信来源
- 合法状态可枚举
- 状态转换有明确入口
- 异步任务可取消或忽略过期结果
- 副作用与纯状态计算分离

### 14.4 架构决策记录

重要技术决策可用轻量 ADR 记录：

```text
标题：选择何种页面状态管理方案
背景：当前问题和约束
决策：最终采用的方案
备选：评估过的其他方案
影响：收益、成本和风险
状态：提议 / 已采用 / 已废弃
```

ADR 记录“为什么”，代码记录“怎么做”。约束变化时可以重新评估，而不是把历史方案当成永久规则。

### 14.5 评审检查

| 维度 | 检查问题 |
| --- | --- |
| 职责 | 类型和模块是否只有清晰的变化原因 |
| 边界 | 外部依赖是否被接口隔离 |
| 依赖 | 是否存在反向、跨层或循环依赖 |
| 状态 | 是否有多个数据源或非法状态组合 |
| 并发 | 线程归属、取消和数据竞争是否明确 |
| 错误 | 错误是否可分类、恢复和观测 |
| 测试 | 核心逻辑能否脱离 UI 和网络测试 |
| 演进 | 新增同类需求是否要修改大量旧代码 |
| 性能 | 抽象是否引入不必要的转换和调用成本 |
| 运维 | 日志、埋点、灰度和回滚是否可用 |

### 14.6 渐进式演进

| 阶段 | 常见方案 |
| --- | --- |
| 简单页面 | MVC 或轻量 SwiftUI 状态 |
| 状态复杂 | ViewModel + 明确页面 State |
| 业务复杂 | UseCase + Repository |
| 流程复杂 | Coordinator / Router |
| 团队扩大 | Feature 模块化 + 接口治理 |
| 多实现并存 | 依赖倒置 + Composition Root |

架构升级应同时删除被替代的旧路径。新旧数据流长期并存，会让“渐进式改造”变成永久双轨。

---

## 15. 完整业务模块示例：订单详情

下面用“加载订单详情并取消订单”说明 Presentation、Domain、Data、Navigation 如何协作。

### 15.1 需求与业务规则

需求：

- 进入页面后加载订单
- 加载中、成功、空数据和失败状态可区分
- 只有待支付订单可以取消
- 取消成功后刷新页面
- 网络实现可以被 Mock 或本地实现替换

业务规则不应散落在按钮点击和网络回调中。

### 15.2 目录结构

```text
OrderDetail/
├── Domain/
│   ├── Order.swift
│   ├── OrderRepository.swift
│   ├── LoadOrderUseCase.swift
│   └── CancelOrderUseCase.swift
├── Data/
│   ├── OrderDTO.swift
│   ├── OrderAPI.swift
│   └── DefaultOrderRepository.swift
├── Presentation/
│   ├── OrderDetailState.swift
│   ├── OrderDetailViewModel.swift
│   └── OrderDetailView.swift
├── Navigation/
│   └── OrderDetailCoordinator.swift
└── OrderDetailAssembly.swift
```

依赖方向：

```text
Presentation ──→ Domain ←── Data
      ↑
Navigation / Assembly
```

Domain 不知道 URLSession、SwiftUI 或 UIKit。

### 15.3 Domain Model

```swift
struct Order: Equatable, Sendable {
    let id: String
    let title: String
    let amount: Decimal
    let status: Status

    enum Status: Equatable, Sendable {
        case pendingPayment
        case paid
        case cancelled
    }

    var canCancel: Bool {
        status == .pendingPayment
    }
}
```

`canCancel` 是领域规则，放在 Entity 或 UseCase，而不是由 View 根据字符串判断。

### 15.4 Repository 抽象

```swift
protocol OrderRepository: Sendable {
    func order(id: String) async throws -> Order
    func cancelOrder(id: String) async throws
}
```

Repository 描述 Domain 需要的数据能力，不暴露 HTTP Method、DTO 或数据库表。

### 15.5 UseCase

```swift
struct LoadOrderUseCase: Sendable {
    private let repository: any OrderRepository

    init(repository: any OrderRepository) {
        self.repository = repository
    }

    func execute(id: String) async throws -> Order {
        guard !id.isEmpty else {
            throw OrderError.invalidID
        }
        return try await repository.order(id: id)
    }
}

struct CancelOrderUseCase: Sendable {
    private let repository: any OrderRepository

    init(repository: any OrderRepository) {
        self.repository = repository
    }

    func execute(order: Order) async throws {
        guard order.canCancel else {
            throw OrderError.cannotCancel
        }
        try await repository.cancelOrder(id: order.id)
    }
}
```

UseCase 负责业务动作、参数校验和规则编排。单纯转发且没有演进可能的场景，不必强制为每个 Repository 方法创建 UseCase。

### 15.6 DTO 与映射

```swift
struct OrderDTO: Decodable {
    let id: String
    let title: String
    let amount: Decimal
    let status: String
}

extension OrderDTO {
    func toDomain() throws -> Order {
        let domainStatus: Order.Status
        switch status {
        case "pending":
            domainStatus = .pendingPayment
        case "paid":
            domainStatus = .paid
        case "cancelled":
            domainStatus = .cancelled
        default:
            throw OrderError.invalidResponse
        }

        return Order(
            id: id,
            title: title,
            amount: amount,
            status: domainStatus
        )
    }
}
```

DTO 跟随后端协议，Domain Model 跟随业务语言。两者分离可避免后端字段直接污染整个 App。

### 15.7 Repository 实现

```swift
struct DefaultOrderRepository: OrderRepository {
    let api: any OrderAPI

    func order(id: String) async throws -> Order {
        let dto = try await api.fetchOrder(id: id)
        return try dto.toDomain()
    }

    func cancelOrder(id: String) async throws {
        try await api.cancelOrder(id: id)
    }
}
```

如果需要缓存，可以用装饰器组合：

```swift
struct CachedOrderRepository: OrderRepository {
    let remote: any OrderRepository
    let cache: OrderCache

    func order(id: String) async throws -> Order {
        if let cached = await cache.order(id: id) {
            return cached
        }
        let order = try await remote.order(id: id)
        await cache.save(order)
        return order
    }

    func cancelOrder(id: String) async throws {
        try await remote.cancelOrder(id: id)
        await cache.remove(id: id)
    }
}
```

### 15.8 页面状态

```swift
enum OrderDetailState: Equatable {
    case idle
    case loading
    case loaded(Order, isCancelling: Bool)
    case failed(message: String)
}
```

使用枚举避免以下非法组合：

```text
isLoading = true
order != nil
error != nil
isCancelling = true
```

页面可以明确决定每种状态如何渲染。

### 15.9 ViewModel

```swift
@MainActor
@Observable
final class OrderDetailViewModel {
    private(set) var state: OrderDetailState = .idle

    private let orderID: String
    private let loadOrder: LoadOrderUseCase
    private let cancelOrder: CancelOrderUseCase
    private var loadTask: Task<Void, Never>?

    init(
        orderID: String,
        loadOrder: LoadOrderUseCase,
        cancelOrder: CancelOrderUseCase
    ) {
        self.orderID = orderID
        self.loadOrder = loadOrder
        self.cancelOrder = cancelOrder
    }

    func start() {
        loadTask?.cancel()
        loadTask = Task { [weak self] in
            await self?.reload()
        }
    }

    func reload() async {
        state = .loading
        do {
            let order = try await loadOrder.execute(id: orderID)
            try Task.checkCancellation()
            state = .loaded(order, isCancelling: false)
        } catch is CancellationError {
            return
        } catch {
            state = .failed(message: error.localizedDescription)
        }
    }

    func cancel() async {
        guard case let .loaded(order, _) = state else { return }

        state = .loaded(order, isCancelling: true)
        do {
            try await cancelOrder.execute(order: order)
            await reload()
        } catch {
            state = .loaded(order, isCancelling: false)
        }
    }

    deinit {
        loadTask?.cancel()
    }
}
```

ViewModel 在主 Actor 上管理页面状态，负责异步任务生命周期，但不负责 HTTP 映射和具体导航。

### 15.10 View

```swift
struct OrderDetailView: View {
    @State private var viewModel: OrderDetailViewModel

    init(viewModel: OrderDetailViewModel) {
        _viewModel = State(initialValue: viewModel)
    }

    var body: some View {
        content
            .task { viewModel.start() }
    }

    @ViewBuilder
    private var content: some View {
        switch viewModel.state {
        case .idle, .loading:
            ProgressView()
        case let .loaded(order, isCancelling):
            VStack {
                Text(order.title)
                Text(order.amount, format: .currency(code: "CNY"))
                if order.canCancel {
                    Button("取消订单") {
                        Task { await viewModel.cancel() }
                    }
                    .disabled(isCancelling)
                }
            }
        case let .failed(message):
            ContentUnavailableView(
                "加载失败",
                systemImage: "exclamationmark.triangle",
                description: Text(message)
            )
        }
    }
}
```

View 只做状态渲染和意图转发，不知道数据来源。

### 15.11 依赖装配

```swift
enum OrderDetailAssembly {
    @MainActor
    static func make(
        orderID: String,
        apiClient: APIClient
    ) -> OrderDetailView {
        let api = DefaultOrderAPI(client: apiClient)
        let repository = DefaultOrderRepository(api: api)
        let loadOrder = LoadOrderUseCase(repository: repository)
        let cancelOrder = CancelOrderUseCase(repository: repository)
        let viewModel = OrderDetailViewModel(
            orderID: orderID,
            loadOrder: loadOrder,
            cancelOrder: cancelOrder
        )
        return OrderDetailView(viewModel: viewModel)
    }
}
```

Assembly 是 Composition Root。对象创建集中在边界，业务对象内部不直接读取全局单例。

### 15.12 单元测试

```swift
actor OrderRepositorySpy: OrderRepository {
    var result: Result<Order, Error>
    private(set) var cancelledIDs: [String] = []

    init(result: Result<Order, Error>) {
        self.result = result
    }

    func order(id: String) async throws -> Order {
        try result.get()
    }

    func cancelOrder(id: String) async throws {
        cancelledIDs.append(id)
    }
}

func testCancelRejectsPaidOrder() async {
    let paidOrder = Order(
        id: "1",
        title: "会员",
        amount: 30,
        status: .paid
    )
    let repository = OrderRepositorySpy(result: .success(paidOrder))
    let useCase = CancelOrderUseCase(repository: repository)

    await #expect(throws: OrderError.cannotCancel) {
        try await useCase.execute(order: paidOrder)
    }
}
```

测试聚焦业务行为，不需要启动页面或真实网络。

---

## 16. 错误模型

错误至少分为三层：

```text
Transport Error
  ├── timeout
  ├── offline
  └── HTTP status
        ↓ 映射
Domain Error
  ├── orderNotFound
  ├── cannotCancel
  └── unauthorized
        ↓ 映射
Presentation Error
  ├── 重试提示
  ├── 登录引导
  └── 业务文案
```

原则：

- 底层错误保留诊断信息
- Domain 不依赖 URLSession 错误类型
- 用户文案由 Presentation 或本地化层决定
- 是否重试由错误类型、幂等性和业务规则共同决定
- 不要用一个 `unknown` 覆盖所有失败

---

## 17. 并发与数据一致性

架构需要明确“谁拥有可变数据”：

```text
View State       → MainActor
Memory Cache     → actor
Database         → 专用上下文 / actor
Network Request  → 可取消 Task
```

典型竞态：

1. 用户先加载 A，再快速切换到 B
2. B 请求先返回，页面显示 B
3. A 请求后返回，错误覆盖成 A

解决方式：

- 切换参数时取消旧 Task
- 提交结果前检查取消状态
- 给请求附加 generation/request ID
- 让状态更新集中在同一 Actor

```swift
private var generation = 0

func load(id: String) async {
    generation += 1
    let current = generation
    let result = await fetch(id)
    guard current == generation else { return }
    apply(result)
}
```

取消是协作式行为，底层任务必须检查取消或使用支持取消的 API。

---

## 18. 模块通信方式

| 方式 | 适用场景 | 风险 |
| --- | --- | --- |
| 直接接口调用 | 同步、明确的能力依赖 | 容易产生反向依赖 |
| Delegate / Closure | 一对一事件回传 | 生命周期管理 |
| Coordinator | 页面和业务流程编排 | Coordinator 膨胀 |
| Notification | 多接收者、弱耦合系统事件 | 类型弱、来源难追踪 |
| Event Bus | 跨模块领域事件 | 隐式依赖和调试困难 |
| Shared Store | 多页面共享状态 | 边界扩大、更新风暴 |

优先使用显式接口。只有真正的一对多系统事件才考虑通知或事件总线，并定义强类型事件、发送者和生命周期。

领域事件示例：

```swift
enum OrderEvent: Sendable {
    case cancelled(orderID: String)
    case paid(orderID: String)
}
```

事件表示已经发生的事实，命令表示希望执行的动作，两者不要混用。

---

## 19. 缓存架构

常见策略：

| 策略 | 行为 | 适用 |
| --- | --- | --- |
| Cache First | 有缓存直接返回 | 低频变化数据 |
| Network First | 网络失败再读缓存 | 时效性较高 |
| Stale While Revalidate | 先返回缓存，再后台刷新 | 内容流、首页 |
| Cache Only | 只读本地 | 离线数据 |

缓存设计必须回答：

- Key 如何构造
- 数据多久过期
- 用户切换时如何隔离
- 写入失败是否影响主请求
- 多级缓存如何保持一致
- Schema 升级如何迁移
- 敏感数据是否允许落盘

缓存不是 Repository 的同义词。Repository 决定业务需要的数据来自哪里，Cache 只是其中一种数据源。

---

## 20. 架构测试策略

```text
          E2E
       Integration
      Component Tests
       Unit Tests
```

| 层 | 测试内容 |
| --- | --- |
| Entity / Value Object | 不变量和计算属性 |
| UseCase | 业务规则和协作流程 |
| Repository | DTO 映射、缓存策略、错误转换 |
| ViewModel | 输入事件到页面状态的转换 |
| View | 关键状态渲染与交互 |
| Coordinator | 路由选择与流程分支 |
| E2E | 登录、支付、订单等核心链路 |

Mock 不应复制真实实现。它只需要提供测试所需行为并记录交互。对于复杂协议，可以提供通用 Fake 或内存实现减少脆弱 Mock。

---

## 21. 架构选型流程

```text
1. 明确业务规模、变化速度和团队边界
2. 找出最难测试、最常变化的部分
3. 定义状态所有权和依赖方向
4. 选择满足约束的最小架构
5. 用一个垂直业务切片验证
6. 测量开发、测试、性能和维护成本
7. 形成模板和自动化约束
8. 定期删除无收益的抽象
```

不要只按团队人数或代码行数选架构。支付规则简单但风险高，可能比代码量更大的展示模块更需要严格分层。

---

## 22. 最终总结

架构的核心不是模式数量，而是四件事：

1. 职责是否清晰
2. 依赖是否单向
3. 状态是否有唯一所有者
4. 变化是否被限制在合理边界

MVC、MVVM、Clean Architecture、TCA 和 Coordinator 都只是实现这些目标的工具。应以业务复杂度、团队协作、测试要求和演进成本为依据选择，并通过完整业务切片验证设计是否真正可用。
