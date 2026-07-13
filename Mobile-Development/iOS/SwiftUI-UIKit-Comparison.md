# SwiftUI 与 UIKit 关系、区别和选型

## 1. SwiftUI 的定位

SwiftUI 是 Apple 推出的声明式 UI 框架，用 Swift 语言描述界面结构、状态和交互。它并不是 UIKit 的简单替代语法，而是一套新的 UI 编程模型。

UIKit 是命令式 UI 框架，开发者直接创建和维护对象；SwiftUI 是声明式 UI 框架，开发者描述状态对应的界面，系统根据状态变化计算和更新 UI。

```text
UIKit
  开发者创建 UIView / UIViewController
  开发者修改属性
  开发者控制生命周期和更新时机

SwiftUI
  开发者声明 View = f(State)
  状态变化触发 body 重新计算
  框架计算差异并更新底层显示
```

二者并不是完全割裂的关系。SwiftUI 在 Apple 平台上运行时仍然依赖系统已有的窗口、事件、渲染、辅助功能和平台控件能力；在 iOS 上，SwiftUI 可以与 UIKit 互相嵌入。

---

## 2. UIKit 与 SwiftUI 的关系

### 2.1 分层关系

从开发者视角看：

```text
App 业务代码
  ├── SwiftUI View
  │     └── 状态驱动 UI
  └── UIKit ViewController / UIView
        └── 对象驱动 UI

系统 UI 基础设施
  ├── 事件分发
  ├── 布局与显示
  ├── Core Animation
  ├── Accessibility
  └── Window / Scene 生命周期
```

SwiftUI 不是简单地把每个 `View` 一一映射成 `UIView`。SwiftUI 的 `View` 通常是轻量值类型，用来描述界面；真正的承载、布局、渲染和事件桥接由框架内部管理。

### 2.2 互操作方式

UIKit 中嵌入 SwiftUI：

```swift
let view = ProfileView(userID: userID)
let controller = UIHostingController(rootView: view)
navigationController.pushViewController(controller, animated: true)
```

SwiftUI 中嵌入 UIKit：

```swift
struct LegacyMapView: UIViewRepresentable {
    func makeUIView(context: Context) -> MKMapView {
        MKMapView(frame: .zero)
    }

    func updateUIView(_ uiView: MKMapView, context: Context) {
        // 根据 SwiftUI state 更新 UIKit view
    }
}
```

嵌入 `UIViewController`：

```swift
struct LegacyEditorController: UIViewControllerRepresentable {
    func makeUIViewController(context: Context) -> EditorViewController {
        EditorViewController()
    }

    func updateUIViewController(_ controller: EditorViewController, context: Context) {
        // 同步外部状态
    }
}
```

---

## 3. 编程模型区别

### 3.1 命令式 vs 声明式

UIKit 关注“怎么改 UI”：

```swift
label.text = user.name
avatarView.image = image
button.isEnabled = user.canFollow
```

SwiftUI 关注“当前状态应该显示什么”：

```swift
struct ProfileHeader: View {
    let user: User

    var body: some View {
        VStack {
            Image(uiImage: user.avatar)
            Text(user.name)
            Button("Follow") {
                // action
            }
            .disabled(!user.canFollow)
        }
    }
}
```

差异本质：

| 维度 | UIKit | SwiftUI |
| --- | --- | --- |
| UI 表达 | 对象树 | View 描述值 |
| 更新方式 | 手动修改对象属性 | 状态变化驱动刷新 |
| 状态管理 | 开发者自行组织 | 属性包装器和数据流约束 |
| 生命周期 | ViewController 生命周期明确 | View 值频繁重建，生命周期更抽象 |
| 调试重点 | 对象引用、回调、约束 | 状态来源、body 计算、依赖变化 |

### 3.2 对象身份 vs 值描述

UIKit 的 `UIView` 和 `UIViewController` 有稳定对象身份：

```text
ProfileViewController
  ├── nameLabel
  ├── avatarView
  └── followButton
```

SwiftUI 的 `View` 多数是值类型描述：

```text
ProfileView(state)
  ↓ body
VStack / Image / Text / Button 描述树
  ↓ diff
更新内部渲染结构
```

因此 SwiftUI 中不要把 `View` 当成长期持有状态的对象。长期状态应放在 `@State`、`@StateObject`、`@Observable` model、Store 或外部数据源中。

---

## 4. 生命周期区别

### 4.1 UIKit 生命周期

UIKit 的页面生命周期以 `UIViewController` 为核心：

```text
init
  ↓
loadView
  ↓
viewDidLoad
  ↓
viewWillAppear
  ↓
viewDidAppear
  ↓
viewWillDisappear
  ↓
viewDidDisappear
```

适合放置：

- 页面创建
- 子视图组装
- 数据绑定
- 导航栏配置
- 与系统生命周期强相关的逻辑

### 4.2 SwiftUI 生命周期

SwiftUI 的 `View` 是可重复创建的值，`body` 会因为依赖状态变化而重新计算：

```text
State 改变
  ↓
body 重新求值
  ↓
生成新的 View 描述
  ↓
SwiftUI diff / reconcile
  ↓
更新显示和交互
```

常用生命周期入口：

| API | 适合场景 |
| --- | --- |
| `.onAppear` | 视图进入显示层级时触发轻量动作 |
| `.onDisappear` | 视图离开显示层级时清理 |
| `.task` | 绑定异步任务到 View 生命周期 |
| `.onChange` | 响应某个状态变化 |
| `@StateObject` / `@Observable` | 持有页面级状态对象 |

注意点：

- `body` 不是只执行一次，不能在里面放副作用。
- `.onAppear` 可能多次触发，加载逻辑要能处理重复调用。
- 页面级对象不要用普通属性临时创建，避免状态丢失。

---

## 5. 状态管理区别

### 5.1 UIKit 状态

UIKit 常见状态来源：

```text
ViewController
  ├── model
  ├── viewModel
  ├── isLoading
  ├── currentPage
  ├── selectedIndexPath
  └── subview 当前属性
```

UIKit 状态灵活，但容易分散：

- Model 中一份
- ViewModel 中一份
- ViewController 属性中一份
- Cell 或子 View 内部又一份

状态分散后，容易出现 UI 与数据不一致。

### 5.2 SwiftUI 状态

SwiftUI 强调单一数据来源：

| 工具 | 归属 | 场景 |
| --- | --- | --- |
| `@State` | 当前 View 私有 | 局部 UI 状态 |
| `@Binding` | 父级传入 | 子 View 读写父级状态 |
| `@StateObject` | 当前 View 创建并持有 | ObservableObject 页面模型 |
| `@ObservedObject` | 外部传入 | 父级持有对象 |
| `@EnvironmentObject` | 环境注入 | 跨层级共享对象 |
| `@Environment` | 系统或自定义环境值 | 主题、尺寸、权限上下文 |

状态流向：

```text
State / Model
  ↓
View 读取状态
  ↓
用户事件
  ↓
Action / Method
  ↓
修改 State / Model
  ↓
View 自动更新
```

SwiftUI 的优势来自状态流清晰；问题也常来自状态归属错误，例如多个地方各自持有一份看似相同的数据。

---

## 6. 布局区别

### 6.1 UIKit 布局

UIKit 主要使用 frame、Auto Layout 或手写布局：

```swift
NSLayoutConstraint.activate([
    titleLabel.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
    titleLabel.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16),
    titleLabel.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor, constant: 20)
])
```

UIKit 布局特点：

- 约束系统成熟
- 与复杂滚动列表、复用机制结合稳定
- 可精细控制 layout pass
- 约束冲突和动态高度调试成本较高

### 6.2 SwiftUI 布局

SwiftUI 布局由容器和修饰符组合：

```swift
VStack(alignment: .leading, spacing: 12) {
    Text(user.name)
        .font(.headline)

    Text(user.bio)
        .font(.body)
}
.padding()
```

布局过程可理解为：

```text
Parent 提供可用尺寸
  ↓
Child 根据自身规则选择尺寸
  ↓
Parent 根据 alignment / spacing 放置 Child
```

SwiftUI 布局特点：

- 声明简洁，组合能力强
- 动态字体、深色模式、无障碍适配更自然
- 自定义复杂布局需要理解 layout proposal
- 与旧版系统行为差异、列表性能和测量问题需要实测

---

## 7. 渲染与性能区别

### 7.1 UIKit 渲染特点

UIKit 的性能优化常围绕对象树和渲染管线：

- 减少 View / Layer 层级
- 降低 Auto Layout 成本
- 避免主线程图片解码
- 避免离屏渲染
- 控制透明混合
- 列表 Cell 复用和预取

UIKit 的优势是行为明确、工具链成熟、性能问题定位路径清晰。

### 7.2 SwiftUI 渲染特点

SwiftUI 的性能问题常围绕状态和重新计算：

- `body` 依赖范围过大
- 上层状态变化导致大范围 View 重新求值
- `ForEach` 缺少稳定 identity
- 在 `body` 中执行重计算或副作用
- 大型列表中状态对象创建位置不当
- 频繁桥接 UIKit 导致同步成本增加

SwiftUI 并不是“状态变化就重绘整个屏幕”。更准确地说，状态变化会触发相关 `body` 重新求值，框架再根据 identity 和差异更新底层显示结构。但如果状态边界设计差，仍然会造成不必要的计算和更新。

---

## 8. 优劣对比

### 8.1 UIKit 优势

- 生态成熟，历史代码和第三方库丰富
- 行为稳定，边界情况资料多
- `UIViewController` 生命周期清晰
- 复杂导航、容器控制器、转场可控性强
- 复杂列表、富交互、精细手势处理经验成熟
- 适合维护大型遗留项目

### 8.2 UIKit 劣势

- 样板代码多
- UI 和状态容易分散
- ViewController 容易膨胀
- 手动同步 UI 状态容易出错
- 动态类型、深色模式、多状态组合需要更多手工处理
- 预览和快速迭代体验不如 SwiftUI

### 8.3 SwiftUI 优势

- 声明式表达简洁
- 状态驱动 UI，减少手动同步
- 组合小 View 成本低
- 与 Swift 类型系统、Concurrency、Observation 配合自然
- Preview 适合快速迭代组件
- 跨 Apple 平台共享 UI 思路更一致
- 动态字体、深色模式、无障碍、环境值适配更顺手

### 8.4 SwiftUI 劣势

- 对最低系统版本要求更敏感
- 某些复杂控件和底层能力仍需 UIKit/AppKit 桥接
- 生命周期抽象后，副作用管理容易踩坑
- 性能问题不总是能从对象树直观看出
- 复杂导航和深链需要团队统一模式
- 不同系统版本行为差异需要更多真机验证

---

## 9. 混编实践

### 9.1 UIKit 项目渐进引入 SwiftUI

适合先从低风险区域开始：

- 设置页
- 空状态页
- 弹窗内容
- 表单页
- 活动页
- 独立业务新页面

推荐路径：

```text
现有 UIKit App
  ↓
新增 SwiftUI 组件
  ↓ UIHostingController 嵌入
  ↓
新页面使用 SwiftUI
  ↓
核心复杂旧页面按收益逐步迁移
```

不要为了迁移而迁移。对于稳定、复杂、依赖大量 UIKit 定制的页面，保持 UIKit 可能更划算。

### 9.2 SwiftUI 项目使用 UIKit 能力

需要 UIKit 桥接的常见场景：

- 系统控件 SwiftUI 暂无完整能力
- 复用已有 UIKit 组件
- 地图、相机、富文本、WebView 等复杂组件
- 特殊手势或转场
- 需要成熟第三方 UIKit SDK

桥接建议：

- 把 UIKit 组件包装成边界清晰的 `Representable`
- SwiftUI state 作为输入，Coordinator 处理 delegate 输出
- 避免 UIKit 内部再维护一套与 SwiftUI 冲突的状态
- 不要在 `updateUIView` / `updateUIViewController` 中无条件重建重对象

---

## 10. 选型建议

| 场景 | 推荐 |
| --- | --- |
| 新项目且最低系统版本允许 | 优先 SwiftUI |
| 大型存量 UIKit 项目 | UIKit 为主，SwiftUI 渐进引入 |
| 高度定制复杂列表 | UIKit 或先做 SwiftUI 性能验证 |
| 表单、设置、详情、状态页 | SwiftUI |
| 复杂导航、深链、多流程复用 | 两者都可，但需要明确 Router/Coordinator |
| 大量第三方 UIKit SDK | UIKit 或 SwiftUI + UIKit 桥接 |
| 团队 UIKit 经验强、SwiftUI 经验弱 | 先在低风险模块试点 SwiftUI |
| 多 Apple 平台共享 UI | SwiftUI 更有优势 |

决策原则：

- 不只看技术新旧，要看系统版本、团队经验和页面复杂度。
- SwiftUI 更适合状态清晰、组合明显、快速迭代的 UI。
- UIKit 更适合强控制、强兼容、深度定制和大量历史资产场景。
- 混编是长期可行路线，但要控制状态边界和生命周期边界。

---

## 11. 常见误区

1. **SwiftUI 会完全替代 UIKit 吗？**  
   对新功能 SwiftUI 的使用会越来越多，但 UIKit 仍然是大量 iOS 应用、系统能力和历史生态的重要基础。实际项目中长期混编很常见。

2. **SwiftUI 性能一定比 UIKit 好？**  
   不一定。SwiftUI 减少了手动更新代码，但状态边界设计不当会带来额外计算。复杂列表、频繁刷新和桥接场景仍需实测。

3. **SwiftUI 不需要架构吗？**  
   需要。声明式 UI 只是减少 UI 同步代码，业务逻辑、数据流、导航、依赖注入仍然需要清晰设计。

4. **UIKit 项目是否应该整体重写成 SwiftUI？**  
   通常不建议。更稳妥的方式是新页面使用 SwiftUI，旧页面按维护成本和收益逐步替换。

5. **SwiftUI 的 ViewModel 能直接操作 UIKit View 吗？**  
   不建议。ViewModel 应表达状态和动作，UIKit/SwiftUI 具体更新应留在 View 层或桥接层。

---

## 12. 总结

UIKit 和 SwiftUI 的核心差异不是语法，而是 UI 编程模型：UIKit 以对象和命令式更新为中心，SwiftUI 以状态和声明式描述为中心。

UIKit 的优势是成熟、可控、兼容性强；SwiftUI 的优势是简洁、状态驱动、组合效率高。实际工程中，最稳妥的策略通常不是二选一，而是在明确边界的前提下混合使用：稳定复杂资产继续用 UIKit，新功能和适合声明式表达的页面优先使用 SwiftUI。
