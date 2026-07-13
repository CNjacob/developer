# iOS 性能优化

## 1. 性能优化概述

iOS 性能优化不是单点技巧，而是围绕用户体验、系统资源和工程规模的一套持续治理能力。常见方向包括：

- 启动优化
- 包体积优化
- 内存占用优化
- UI 渲染性能优化
- 滚动流畅度优化
- CPU 占用优化
- I/O 优化
- 网络性能优化
- 耗电和发热优化
- 性能监控和回归治理

性能优化的基本原则：

1. 先测量，再优化。
2. 优先解决用户可感知问题。
3. 区分 Debug、Release、真机、模拟器差异。
4. 优化要可验证、可回归、可长期监控。
5. 不为局部指标牺牲架构清晰度和稳定性。

常见性能指标：

| 指标 | 含义 | 用户感知 |
| --- | --- | --- |
| 启动耗时 | 点击图标到首屏可交互时间 | App 打开快慢 |
| FPS / 卡顿率 | 页面滚动和动画流畅度 | 掉帧、卡顿 |
| CPU 占用 | 计算资源消耗 | 发热、卡顿、耗电 |
| 内存峰值 | 运行时内存使用 | 被系统杀进程、OOM |
| 包体积 | 下载包和安装后体积 | 下载转化、安装空间 |
| I/O 耗时 | 文件和数据库读写耗时 | 启动慢、页面慢 |
| 网络耗时 | DNS、连接、TLS、请求响应 | 首屏慢、接口慢 |

---

## 2. iOS App 启动流程

启动优化必须先理解完整启动链路。iOS App 启动大致可以分为：

```text
用户点击图标
  ↓
SpringBoard 发起启动请求
  ↓
系统创建进程
  ↓
加载 Mach-O 可执行文件
  ↓
dyld 加载动态库和 Framework
  ↓
ObjC Runtime 注册类、分类、协议
  ↓
执行 +load、C++ 静态构造、constructor
  ↓
进入 main()
  ↓
UIApplicationMain
  ↓
创建 UIApplication / AppDelegate / SceneDelegate
  ↓
创建首屏 UI
  ↓
首帧渲染提交
  ↓
用户可交互
```

从优化角度，启动通常分为三个阶段：

| 阶段 | 范围 | 可控性 |
| --- | --- | --- |
| pre-main | 进 main 之前 | 部分可控 |
| main 到首屏创建 | `main()` 到首屏 View 创建 | 高 |
| 首屏渲染到可交互 | 首屏布局、渲染、数据加载 | 高 |

---

## 3. pre-main 阶段原理

pre-main 是 `main()` 执行前的阶段，主要由系统、dyld、Runtime 完成。

### 3.1 Mach-O 加载

iOS App 的可执行文件是 Mach-O 格式。Mach-O 包含：

- Header
- Load Commands
- `__TEXT` 段
- `__DATA` 段
- `__LINKEDIT` 段
- 符号表
- 重定位信息
- 依赖动态库信息

启动时系统会把 Mach-O 映射到进程地址空间。

```text
Mach-O
├── __TEXT
│   ├── 代码
│   ├── 常量字符串
│   └── 只读数据
├── __DATA
│   ├── 全局变量
│   ├── 静态变量
│   └── 懒加载符号指针
└── __LINKEDIT
    ├── 符号表
    ├── 字符串表
    └── 重定位信息
```

### 3.2 dyld 加载动态库

dyld 是 Apple 的动态链接器。它负责加载 App 可执行文件依赖的动态库和 Framework。

简化流程：

```text
dyld 启动
  ↓
读取 Mach-O Load Commands
  ↓
加载依赖动态库
  ↓
递归加载依赖库的依赖
  ↓
Rebase
  ↓
Bind
  ↓
ObjC Runtime 初始化
  ↓
执行初始化函数
  ↓
调用 main()
```

动态库越多，dyld 需要处理的依赖、符号绑定、初始化工作越多，pre-main 耗时通常越高。

### 3.3 Rebase

由于 ASLR（地址空间布局随机化），Mach-O 加载地址每次可能不同。Rebase 的作用是修正镜像内部指针。

```text
编译期地址
  ↓ 加载地址发生偏移
运行期真实地址 = 编译期地址 + slide
```

需要修正的指针越多，Rebase 成本越高。

### 3.4 Bind

Bind 的作用是把当前 Mach-O 中引用的外部符号绑定到真实地址。

例如 App 调用某个系统库函数：

```objc
NSLog(@"Hello");
```

dyld 需要找到 `NSLog` 所在动态库中的真实地址，并写入符号指针表。

影响 Bind 成本的因素：

- 动态库数量
- 外部符号数量
- Objective-C 类和分类数量
- C++ 虚表和全局符号

### 3.5 ObjC Runtime 初始化

dyld 加载 Mach-O 后，Objective-C Runtime 会读取并注册：

- 类
- 分类
- 协议
- selector
- method list
- property list
- ivar layout

```text
读取 __objc_classlist
  ↓
注册 Class
  ↓
读取 __objc_catlist
  ↓
把 Category 方法附加到目标类
  ↓
注册 Protocol / Selector
```

类、分类、方法越多，Runtime 初始化成本越高。

### 3.6 Category 加载成本

Category 会在启动时被 Runtime 处理：

```text
找到 Category
  ↓
找到目标 Class
  ↓
合并方法列表、协议列表、属性列表
  ↓
必要时清理方法缓存
```

大量 Category，尤其是对系统类的大量 Category，会增加启动期 Runtime 工作量。

### 3.7 `+load`

`+load` 在类和分类加载时执行，早于 `main()`。

```objc
+ (void)load {
    [Tracker registerPage:@"Home"];
}
```

问题：

- 所有 `+load` 都会在启动期执行
- 执行顺序复杂
- 容易引入隐式依赖
- 任何耗时操作都会直接增加 pre-main

`+load` 中不应做：

- 网络请求
- 数据库初始化
- 大文件读取
- 复杂对象创建
- 大量注册逻辑
- 同步锁等待

### 3.8 C++ 静态初始化和 constructor

这些也发生在 `main()` 前：

```objc
__attribute__((constructor))
static void setup() {
    // main 前执行
}
```

C++ 全局对象构造也会在启动前执行。过多静态初始化会增加 pre-main 耗时。

### 3.9 静态库与动态库

| 类型 | 特点 | 启动影响 |
| --- | --- | --- |
| 静态库 | 编译链接进主 Mach-O | 减少动态库加载数量，但可能增大主二进制 |
| 动态库 | 运行时由 dyld 加载 | 增加加载、绑定、初始化成本 |

动态 Framework 过多是启动慢的常见原因。能合并的业务动态库可以考虑合并，纯工具模块可考虑静态链接。

---

## 4. main 后启动流程

进入 `main()` 后，开发者可控性明显增加。

### 4.1 `UIApplicationMain`

```objc
int main(int argc, char * argv[]) {
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

`UIApplicationMain` 主要做：

- 创建 `UIApplication`
- 创建 AppDelegate
- 设置主 RunLoop
- 加载 Info.plist
- 建立事件循环
- 处理生命周期回调

### 4.2 AppDelegate / SceneDelegate

iOS 13 以后，窗口和场景创建通常在 `SceneDelegate`：

```swift
func scene(
    _ scene: UIScene,
    willConnectTo session: UISceneSession,
    options connectionOptions: UIScene.ConnectionOptions
) {
    let window = UIWindow(windowScene: windowScene)
    window.rootViewController = HomeViewController()
    self.window = window
    window.makeKeyAndVisible()
}
```

这里常见耗时：

- 初始化 SDK
- 创建根控制器
- 构建 TabBar
- 同步读取配置
- 数据库初始化
- 读取用户状态
- 网络请求阻塞 UI

### 4.3 首屏构建

首屏可见前通常经历：

```text
创建 RootViewController
  ↓
loadView
  ↓
viewDidLoad
  ↓
viewWillAppear
  ↓
Auto Layout
  ↓
图片解码 / 文本排版
  ↓
Core Animation 提交
  ↓
首帧显示
```

启动优化不只是让 `main()` 更快，还要让首屏更快显示并可交互。

---

## 5. 启动耗时测量

### 5.1 Xcode 启动诊断

可使用环境变量查看 pre-main：

```text
DYLD_PRINT_STATISTICS=1
DYLD_PRINT_STATISTICS_DETAILS=1
```

输出通常包含：

- dylib loading time
- rebase/binding time
- ObjC setup time
- initializer time
- total pre-main time

### 5.2 手动埋点

`main()` 中记录时间：

```objc
CFAbsoluteTime AppStartTime;

int main(int argc, char * argv[]) {
    AppStartTime = CFAbsoluteTimeGetCurrent();
    @autoreleasepool {
        return UIApplicationMain(argc, argv, nil, NSStringFromClass([AppDelegate class]));
    }
}
```

首屏出现：

```objc
- (void)viewDidAppear:(BOOL)animated {
    [super viewDidAppear:animated];
    CFAbsoluteTime cost = CFAbsoluteTimeGetCurrent() - AppStartTime;
    NSLog(@"launch cost: %.3f", cost);
}
```

### 5.3 冷启动与热启动

| 类型 | 含义 |
| --- | --- |
| 冷启动 | App 进程不存在，从零创建进程 |
| 热启动 | App 已在后台，回到前台 |
| 温启动 | 进程可能存在部分缓存，状态介于两者之间 |

启动优化重点通常是冷启动和首屏可交互。

---

## 6. 启动优化策略

### 6.1 减少动态库数量

优化方向：

- 合并业务动态库
- 能静态链接的库使用静态链接
- 避免引入大量小 Framework
- 清理无用依赖
- 谨慎使用插件化动态库

原则：动态库不是不能用，但不能无节制拆分。

### 6.2 减少 `+load`

替代方案：

- 显式注册
- 懒加载
- `dispatch_once`
- 启动完成后异步初始化
- 编译期生成注册表

不推荐大量组件通过 `+load` 自动注册。

### 6.3 延迟初始化 SDK

启动期只初始化首屏必须能力。非必要 SDK 延后：

- 分享 SDK
- 支付 SDK
- 地图 SDK
- IM SDK
- 广告 SDK
- 埋点部分高级能力

示例：

```swift
func applicationDidFinishLaunching() {
    setupMinimalAnalytics()
    showRootViewController()

    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
        self.setupNonCriticalSDKs()
    }
}
```

### 6.4 首屏懒加载

首屏只创建可见内容：

- TabBar 只创建第一个 Tab，其他 Tab 懒加载
- 首页非首屏模块延迟构建
- 图片异步加载和预解码
- 非关键接口延后请求
- 配置读取异步化

### 6.5 减少主线程 I/O

启动期避免：

- 同步读取大文件
- 同步数据库迁移
- 同步解压资源
- 同步解析大 JSON
- 主线程写日志

### 6.6 首屏数据策略

常见策略：

- 先展示骨架屏
- 先展示缓存，再刷新网络
- 首屏接口聚合
- 关键数据优先
- 非关键数据延迟

---

## 7. 包体积优化

包体积包括下载包大小和安装后大小。影响用户下载转化、安装空间和审核上传效率。

### 7.1 包体积构成

```text
.ipa
├── Mach-O 可执行文件
├── Frameworks
├── Assets.car
├── 图片 / 音频 / 视频
├── Storyboard / XIB
├── 字体
├── 本地化资源
└── 配置文件
```

### 7.2 可执行文件优化

优化方向：

- 开启 Release 优化
- Strip Symbols
- 移除无用代码
- 清理无用三方库
- 避免重复引入同一库
- 使用 Link-Time Optimization
- Swift 泛型和模板代码避免过度膨胀

构建设置常见项：

| 设置 | 作用 |
| --- | --- |
| `Strip Linked Product` | 剥离符号 |
| `Deployment Postprocessing` | 发布后处理 |
| `Dead Code Stripping` | 移除无用代码 |
| `Optimization Level` | 编译优化 |

### 7.3 图片资源优化

图片通常是包体积大头。

优化：

- 删除无用图片
- 使用 Asset Catalog
- 合理选择 PNG/JPEG/WebP/HEIF
- 大图下发而不是内置
- 纯色和简单图形用代码绘制或矢量
- 压缩图片
- 避免 1x/2x/3x 重复无效资源

### 7.4 资源按需下载

不必随包内置所有资源：

- 活动资源
- 教程视频
- 大型动效
- 低频业务素材
- 地图或离线包

可使用：

- On-Demand Resources
- 服务端动态下发
- 首次使用时下载
- CDN 缓存

### 7.5 本地化清理

如果引入三方库带大量语言资源，但产品不支持这些语言，可以考虑清理无用 `.lproj`。

注意：清理要确保不影响系统权限文案、审核和目标市场。

---

## 8. 内存优化

### 8.1 iOS 内存模型

iOS 没有传统 swap。内存压力过高时，系统会：

- 发送内存警告
- 回收缓存
- 终止后台 App
- 严重时杀死当前 App

常见内存指标：

| 指标 | 含义 |
| --- | --- |
| Resident Memory | 实际驻留物理内存 |
| Dirty Memory | App 修改过且不能直接丢弃的内存 |
| Clean Memory | 可从文件重新加载的内存 |
| VM Size | 虚拟地址空间，不等于真实占用 |

优化重点通常是 Dirty Memory 和峰值内存。

### 8.2 图片内存

图片内存按解码后像素计算，不按文件大小计算。

```text
内存 = 宽 × 高 × 4 bytes
4000 × 3000 × 4 ≈ 45.8 MB
```

优化：

- 使用缩略图
- 按显示尺寸解码
- 避免直接加载超大图
- 列表中及时取消图片请求
- 使用内存缓存上限
- 内存警告时清理缓存

### 8.3 循环引用

常见循环引用：

- 对象和 Block
- delegate 强引用
- Timer 强引用 target
- CADisplayLink 强引用 target
- Notification 未移除

Timer 示例：

```swift
timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
    self?.tick()
}
```

### 8.4 AutoreleasePool

大量临时对象循环中要使用 autoreleasepool：

```objc
for (NSInteger i = 0; i < files.count; i++) {
    @autoreleasepool {
        NSData *data = [NSData dataWithContentsOfFile:files[i]];
        [self process:data];
    }
}
```

### 8.5 缓存策略

缓存不是越多越好。要设置：

- 最大数量
- 最大内存
- 过期时间
- 内存警告清理
- 前后台切换清理策略

---

## 9. UI 渲染性能

### 9.1 渲染管线

```text
主线程更新 UI
  ↓
布局计算
  ↓
文本排版 / 图片解码
  ↓
Core Animation 提交 Layer Tree
  ↓
Render Server
  ↓
GPU 合成
  ↓
屏幕显示
```

CPU 和 GPU 任一阶段超过帧预算都会掉帧。

| 刷新率 | 每帧预算 |
| --- | --- |
| 60Hz | 16.67ms |
| 120Hz | 8.33ms |

### 9.2 离屏渲染

离屏渲染会让 GPU 先渲染到额外缓冲区，再合成回来。

常见触发：

- mask
- 动态阴影
- 圆角裁剪叠加复杂内容
- `shouldRasterize`
- 部分滤镜和视觉效果

优化：

```objc
view.layer.shadowPath = [UIBezierPath bezierPathWithRect:view.bounds].CGPath;
```

### 9.3 透明混合

半透明图层需要 GPU 读取下层像素混合。

优化：

- 不透明 View 设置 `opaque = YES`
- Cell 背景使用不透明色
- 减少半透明层级
- 避免大面积透明 PNG

### 9.4 Auto Layout 优化

复杂约束会增加布局成本。

优化：

- 减少约束数量
- 避免频繁创建/销毁约束
- 更新 `constant` 而不是重建约束
- 列表 Cell 预计算高度
- 简单布局可用 frame
- 避免在 `layoutSubviews` 中反复触发布局

### 9.5 文本渲染

文本排版可能很重，尤其是复杂富文本和列表。

优化：

- 后台计算文本高度
- 缓存排版结果
- 控制富文本复杂度
- 避免 Cell 中同步创建复杂 attributed string

---

## 10. 流畅度优化

### 10.1 卡顿原因

卡顿通常来自主线程或渲染线程超时：

- 主线程执行耗时任务
- Auto Layout 复杂
- 图片同步解码
- 文本排版过重
- 数据源处理过重
- 大量 Cell 创建
- GPU 离屏渲染
- 透明混合过多
- 锁等待
- 主线程 I/O

### 10.2 列表优化

列表是流畅度优化重点。

优化清单：

- Cell 复用
- 异步图片加载
- 图片按显示尺寸解码
- 快速 `prepareForReuse`
- 取消不可见 Cell 请求
- 预计算高度
- 减少圆角、阴影、透明
- 数据 diff 计算放后台
- 主线程只做 UI 更新

异步图片校验：

```swift
let id = model.id
cell.representedID = id

imageLoader.load(model.url) { image in
    DispatchQueue.main.async {
        guard cell.representedID == id else { return }
        cell.imageView.image = image
    }
}
```

### 10.3 预加载与预渲染

可以提前做：

- 图片下载
- 图片解码
- Cell 高度计算
- 文本排版
- 数据请求

但预加载要控制范围，避免 CPU、内存、网络反而变高。

### 10.4 卡顿监控

常见方案：

- FPS 监控
- RunLoop Observer
- 主线程堆栈采样
- MetricKit
- Instruments

RunLoop 卡顿监控只能说明主线程长时间没有进入休眠或被唤醒，不能直接定位根因，通常要结合堆栈采样。

---

## 11. CPU 占用优化

### 11.1 CPU 高的原因

常见原因：

- 主线程复杂计算
- JSON 解析过大
- 图片解码
- 加密压缩
- 数据库查询
- 频繁日志
- 死循环或高频 Timer
- 锁竞争
- 过多线程上下文切换
- Swift 泛型/协议抽象在热路径开销过高

### 11.2 优化策略

- 耗时任务移到后台队列
- 分批处理大任务
- 使用缓存减少重复计算
- 降低 Timer 频率
- 合并短时间重复任务
- 避免主线程锁等待
- 使用 Instruments Time Profiler 定位热点
- 热路径减少动态派发和桥接

示例：分批处理，避免长时间占用主线程：

```swift
func processInBatches(_ items: [Item], batchSize: Int = 100) {
    var index = 0

    func nextBatch() {
        let end = min(index + batchSize, items.count)
        process(items[index..<end])
        index = end

        if index < items.count {
            DispatchQueue.main.async {
                nextBatch()
            }
        }
    }

    nextBatch()
}
```

### 11.3 线程不是越多越好

线程过多会带来：

- 上下文切换
- 内存栈开销
- 锁竞争
- 调度成本
- CPU cache miss

推荐使用：

- GCD 队列
- OperationQueue 限制并发数
- Swift Concurrency 结构化任务
- 专用串行队列保护资源

---

## 12. I/O 优化

### 12.1 常见 I/O 问题

- 启动期同步读大文件
- 主线程读写数据库
- 频繁小文件读写
- 日志同步写入
- 大量 UserDefaults 写入
- 解压缩阻塞主线程

### 12.2 优化策略

- I/O 放后台
- 合并写入
- 使用缓存
- 控制日志频率
- 大文件流式读取
- 数据库查询加索引
- 避免启动期数据库迁移

UserDefaults 不适合存大数据：

```swift
UserDefaults.standard.set(largeJSONData, forKey: "data") // 不推荐
```

### 12.3 数据库优化

常见策略：

- 建索引
- 分页查询
- 避免主线程查询
- 使用事务批量写入
- 控制字段数量
- 大字段拆分

---

## 13. 网络性能优化

### 13.1 网络耗时拆解

```text
DNS
  ↓
TCP connect
  ↓
TLS handshake
  ↓
Request upload
  ↓
Server processing
  ↓
Response download
  ↓
Decode / render
```

### 13.2 优化策略

- DNS 缓存或 HTTPDNS
- 连接复用
- HTTP/2 或 HTTP/3
- 接口合并
- 数据压缩
- 图片按需裁剪
- 缓存策略
- 预请求
- 超时和重试控制

首屏接口要优先级更高，非关键接口延后。

---

## 14. 耗电与发热

耗电和发热通常来自：

- CPU 长时间高占用
- GPU 高负载
- 定位持续开启
- 高频网络请求
- 音视频采集编码
- 后台任务滥用
- 传感器频繁读取

优化：

- 降低采样频率
- 按需开启定位
- 合并网络请求
- 降低视频分辨率/帧率
- 使用硬件编解码
- 后台及时停止任务
- 避免高频 Timer

---

## 15. 性能分析工具

| 工具 | 用途 |
| --- | --- |
| Time Profiler | CPU 热点 |
| Allocations | 内存分配 |
| Leaks | 泄漏 |
| VM Tracker | 虚拟内存 |
| Core Animation | FPS、离屏渲染、透明混合 |
| Energy Log | 耗电 |
| Network | 网络请求 |
| MetricKit | 线上性能指标 |
| Xcode Organizer | 崩溃和性能报告 |
| os_signpost | 自定义性能区间 |

`os_signpost` 示例：

```swift
import os

let log = OSLog(subsystem: "com.example.app", category: "launch")
let signpostID = OSSignpostID(log: log)

os_signpost(.begin, log: log, name: "HomeLoad", signpostID: signpostID)
loadHome()
os_signpost(.end, log: log, name: "HomeLoad", signpostID: signpostID)
```

---

## 16. 性能监控与治理

线上性能优化需要长期监控：

- 启动耗时 P50/P90/P99
- 页面打开耗时
- 卡顿率
- OOM 率
- CPU 高占用次数
- 内存峰值
- 网络成功率和耗时
- 包体积趋势

治理建议：

- 建立性能基线
- CI 中监控包体积
- 关键页面做自动化性能测试
- 每次大版本对比启动和内存
- 性能指标按设备型号和系统版本拆分
- 线上异常要能回溯版本和功能变更

---

## 17. 常见优化误区

- 没有测量就优化。
- 只看平均值，不看 P90/P99。
- 只优化 Debug，不测 Release。
- 只在高端机测试。
- 把所有初始化都丢到后台但不控制时机。
- 为了启动快牺牲首屏可交互。
- 缓存无限增长。
- 线程开得越多越好。
- 静默吞掉性能异常，不做监控。

---

## 18. 面试高频问题

1. **iOS App 启动流程是什么？**  
   点击图标后系统创建进程，加载 Mach-O，dyld 加载动态库，做 rebase/bind，Runtime 注册 ObjC 类和分类，执行 `+load`、constructor，然后进入 `main`、`UIApplicationMain`，创建 AppDelegate/SceneDelegate、首屏 UI，最后完成首帧渲染。

2. **pre-main 主要耗时在哪里？**  
   动态库加载、rebase/bind、ObjC Runtime 初始化、`+load`、C++ 静态初始化和 constructor。

3. **如何优化启动？**  
   减少动态库数量，减少 `+load`，延迟非必要 SDK，首屏懒加载，减少主线程 I/O，先展示缓存或骨架屏。

4. **包体积怎么优化？**  
   清理无用代码和资源，压缩图片，减少无用三方库，Strip 符号，按需下载大资源，清理无用本地化。

5. **图片为什么容易导致内存暴涨？**  
   图片显示时按解码后像素占内存，不按文件大小。大图解码后可能几十 MB。

6. **卡顿怎么定位？**  
   先用 FPS/RunLoop/MetricKit 发现问题，再用 Time Profiler、Core Animation、主线程堆栈采样定位 CPU、布局、渲染或 I/O 根因。

7. **CPU 高怎么优化？**  
   找热点，减少重复计算，移出主线程，分批处理，控制 Timer，减少锁竞争和线程切换，缓存结果。

---

## 19. 总结

iOS 性能优化要从启动、包体积、内存、渲染、流畅度、CPU、I/O、网络和耗电多个维度系统治理。启动优化的关键是理解 Mach-O、dyld、Runtime、`+load`、动态库和首屏构建链路；运行时性能的关键是控制主线程工作量、内存峰值、GPU 压力和后台任务。

真正有效的性能优化不是一次性改几个点，而是建立测量、分析、优化、验证、监控的闭环。
