# HarmonyOS 进阶知识：渲染、运行时、性能、安全与音视频

## 1. 与 iOS 知识点的对应关系

| iOS 主题 | HarmonyOS 对应主题 |
|----------|--------------------|
| UIView / CALayer 渲染 | ArkUI 声明式 UI、组件树、状态驱动刷新、布局与绘制 |
| RunLoop | Ability 生命周期、ArkUI 事件、任务调度、异步 Promise |
| Runtime | ArkTS、Ark Runtime、模块加载、HAR/HSP |
| 性能优化 | 启动、页面渲染、列表、内存、包体积、I/O、网络 |
| 安全机制 | 应用沙箱、权限、签名、证书、数据安全、网络安全 |
| 音视频开发 | AVPlayer、相机、录音、编解码、Surface/媒体组件 |

## 2. ArkUI 渲染与状态驱动

ArkUI 是声明式 UI。页面不是手动查找控件再修改属性，而是通过状态变化驱动 UI 更新。

```text
State
  |
  v
build()
  |
  v
Component Tree
  |
  v
Layout
  |
  v
Render
```

示例：

```ts
@Entry
@Component
struct CounterPage {
  @State count: number = 0

  build() {
    Column() {
      Text(`Count: ${this.count}`)
        .fontSize(24)

      Button('Add')
        .onClick(() => {
          // 修改 @State 后触发相关 UI 刷新
          this.count++
        })
    }
    .width('100%')
    .height('100%')
    .justifyContent(FlexAlign.Center)
  }
}
```

性能要点：

- 状态粒度不要过大，避免牵连大量组件刷新。
- 列表项要提供稳定 key。
- 复杂组件拆分为小组件。
- 图片、网络、文件操作不要阻塞页面构建。
- 不要在 `build()` 中做重计算或发起请求。

## 3. ArkUI 布局与列表性能

列表示例：

```ts
interface User {
  id: number
  name: string
}

@Entry
@Component
struct UserListPage {
  @State users: User[] = [
    { id: 1, name: 'Tom' },
    { id: 2, name: 'Jerry' }
  ]

  build() {
    List() {
      ForEach(this.users, (user: User) => {
        ListItem() {
          UserRow({ user })
        }
      }, (user: User) => user.id.toString())
    }
  }
}

@Component
struct UserRow {
  @Prop user: User

  build() {
    Row() {
      Text(this.user.name)
        .fontSize(18)
    }
    .height(56)
    .padding({ left: 16, right: 16 })
  }
}
```

要点：

- `ForEach` 使用稳定 key。
- 列表项组件保持轻量。
- 图片异步加载和缓存。
- 避免嵌套过深。
- 分页加载，避免一次性渲染过多数据。

## 4. Ability 生命周期与任务调度

UIAbility 生命周期：

```text
onCreate
  |
  v
onWindowStageCreate
  |
  v
onForeground
  |
  v
onBackground
  |
  v
onWindowStageDestroy
  |
  v
onDestroy
```

示例：

```ts
import UIAbility from '@ohos.app.ability.UIAbility'
import window from '@ohos.window'

export default class EntryAbility extends UIAbility {
  onCreate(): void {
    // 只做轻量初始化，避免拖慢启动
    console.info('onCreate')
  }

  onWindowStageCreate(windowStage: window.WindowStage): void {
    windowStage.loadContent('pages/Index', (err) => {
      if (err.code) {
        console.error(`load page failed: ${err.message}`)
      }
    })
  }

  onBackground(): void {
    // 进入后台时保存状态、暂停动画或释放可暂停资源
    console.info('onBackground')
  }
}
```

异步任务：

```ts
async function loadUserName(): Promise<string> {
  // 模拟网络或数据库读取
  return 'Tom'
}

@Entry
@Component
struct UserPage {
  @State name: string = ''

  async aboutToAppear(): Promise<void> {
    try {
      this.name = await loadUserName()
    } catch (e) {
      console.error(`load failed: ${JSON.stringify(e)}`)
    }
  }

  build() {
    Text(this.name)
  }
}
```

注意：

- 页面即将消失时要取消或忽略过期任务结果。
- 大计算不要放在 UI 构建路径。
- 后台任务要符合系统限制和权限要求。

## 5. ArkTS、Ark Runtime 与模块化

HarmonyOS 工程中常见包类型：

| 类型 | 说明 |
|------|------|
| HAP | 应用功能包，安装运行基本单元 |
| HAR | 静态共享包，适合公共组件、工具库 |
| HSP | 动态共享包，适合运行时共享能力 |

模块化理解：

```text
entry HAP
  |
  +-- feature_user HAP/HAR
  +-- common_ui HAR
  +-- common_network HAR
  +-- common_utils HAR
```

公共组件示例：

```ts
@Component
export struct PrimaryButton {
  @Prop text: string
  private onTap: () => void = () => {}

  build() {
    Button(this.text)
      .height(44)
      .width('100%')
      .backgroundColor('#0A59F7')
      .fontColor(Color.White)
      .onClick(() => {
        this.onTap()
      })
  }
}
```

模块化建议：

- 小项目不要过早拆模块。
- 公共能力稳定后再沉淀到 HAR。
- 业务模块不要反向依赖入口模块。
- common 模块不要放业务逻辑。

## 6. 启动性能优化

启动路径：

```text
点击图标
  |
  v
进程创建
  |
  v
AbilityStage 初始化
  |
  v
UIAbility.onCreate
  |
  v
onWindowStageCreate
  |
  v
loadContent
  |
  v
首页首帧
```

优化策略：

- `onCreate` 只做必要初始化。
- SDK 初始化分级延后。
- 首页优先展示静态框架或缓存。
- 网络请求不要阻塞首帧。
- 大对象和大资源懒加载。
- 日志和调试开关在 Release 中收敛。

示例：

```ts
export default class EntryAbility extends UIAbility {
  onCreate(): void {
    // 必需初始化
    this.initCrashReporter()

    // 非关键初始化延后到页面后或用户触发时
  }

  private initCrashReporter(): void {
    console.info('init crash reporter')
  }
}
```

## 7. 页面渲染与流畅度优化

常见问题：

| 问题 | 原因 |
|------|------|
| 页面打开慢 | 首屏加载过多数据或资源 |
| 滚动卡顿 | 列表项复杂、图片过大、无分页 |
| 状态变化卡顿 | 状态粒度过粗，刷新范围过大 |
| 内存上涨 | 缓存不清理、大图过多 |

列表分页示例：

```ts
@Entry
@Component
struct FeedPage {
  @State items: string[] = []
  @State page: number = 1
  @State loading: boolean = false

  async loadMore(): Promise<void> {
    if (this.loading) {
      return
    }

    this.loading = true

    try {
      const newItems = await this.requestPage(this.page)
      this.items = this.items.concat(newItems)
      this.page++
    } finally {
      this.loading = false
    }
  }

  async requestPage(page: number): Promise<string[]> {
    return [`item-${page}-1`, `item-${page}-2`]
  }

  build() {
    List() {
      ForEach(this.items, (item: string) => {
        ListItem() {
          Text(item).height(48)
        }
      }, (item: string) => item)
    }
    .onReachEnd(() => {
      this.loadMore()
    })
  }
}
```

## 8. 内存与资源管理

建议：

- 页面退出时释放监听、定时器、媒体对象。
- 大图按需加载。
- 缓存设置上限。
- 长生命周期对象不要持有页面上下文。
- 使用 Profiler 观察内存曲线。

定时任务释放示例：

```ts
@Entry
@Component
struct TimerPage {
  private timerId: number = -1
  @State count: number = 0

  aboutToAppear(): void {
    this.timerId = setInterval(() => {
      this.count++
    }, 1000)
  }

  aboutToDisappear(): void {
    if (this.timerId >= 0) {
      clearInterval(this.timerId)
      this.timerId = -1
    }
  }

  build() {
    Text(`count=${this.count}`)
  }
}
```

## 9. 包体积优化

包体积来源：

```text
ArkTS 编译产物
资源图片
多语言资源
HAR/HSP 依赖
媒体文件
三方 SDK
```

优化策略：

- 删除无用资源。
- 图片压缩，避免原图打包。
- 公共能力抽到 HAR/HSP，但不要过度依赖。
- 大资源走按需下载。
- Release 关闭调试日志。
- 依赖定期清理。

## 10. HarmonyOS 安全机制

基础安全点：

| 机制 | 说明 |
|------|------|
| 应用沙箱 | 应用数据隔离 |
| 权限模型 | 受保护能力需要声明和授权 |
| 签名证书 | 标识应用来源和完整性 |
| 安全存储 | 敏感数据要加密或使用安全能力 |
| 网络安全 | HTTPS、证书校验、敏感字段保护 |
| 隐私合规 | 权限最小化、用途说明、用户授权 |

权限声明：

```json5
{
  "module": {
    "requestPermissions": [
      {
        "name": "ohos.permission.CAMERA",
        "reason": "$string:camera_reason",
        "usedScene": {
          "abilities": ["EntryAbility"],
          "when": "inuse"
        }
      }
    ]
  }
}
```

安全建议：

- 不在日志打印手机号、Token、身份证、定位。
- 本地敏感数据加密。
- 网络接口必须服务端鉴权。
- Web 容器限制 JSBridge 暴露面。
- 三方 SDK 做权限和隐私审查。

## 11. 网络安全

建议：

- 默认使用 HTTPS。
- 不信任客户端传来的用户身份。
- Token 设置有效期和刷新机制。
- 失败重试要有上限。
- 错误日志不要包含敏感响应体。

请求封装示例：

```ts
export class ApiClient {
  async get<T>(url: string): Promise<T> {
    try {
      // 真实项目中这里调用系统 HTTP 能力或封装库
      const response = await this.request(url)
      return response as T
    } catch (e) {
      console.error(`request failed, url=${url}`)
      throw e
    }
  }

  private async request(url: string): Promise<unknown> {
    return {}
  }
}
```

## 12. 音视频开发

常见能力：

| 场景 | 能力 |
|------|------|
| 音频播放 | 音频播放 API / AVPlayer |
| 视频播放 | 视频组件、AVPlayer |
| 相机拍照 | Camera Kit |
| 录音 | Audio 录制能力 |
| 编解码 | Media Kit 相关能力 |
| 渲染 | XComponent / Surface 类能力，视场景选择 |

视频页面伪代码结构：

```ts
@Entry
@Component
struct VideoPage {
  @State playing: boolean = false

  aboutToDisappear(): void {
    // 页面离开时暂停或释放播放器
    this.stopPlayer()
  }

  private startPlayer(): void {
    this.playing = true
  }

  private stopPlayer(): void {
    this.playing = false
  }

  build() {
    Column() {
      Text(this.playing ? 'Playing' : 'Stopped')

      Button(this.playing ? 'Stop' : 'Play')
        .onClick(() => {
          if (this.playing) {
            this.stopPlayer()
          } else {
            this.startPlayer()
          }
        })
    }
  }
}
```

音视频要点：

- 播放器跟随页面生命周期释放。
- 相机、麦克风必须声明并申请权限。
- 录制前检查存储和隐私授权。
- 直播和实时音视频关注延迟、弱网、丢帧。
- 大文件上传下载要支持进度和取消。

## 13. 调试与性能工具

| 工具 | 用途 |
|------|------|
| Previewer | UI 快速预览 |
| Console / Log | 日志排查 |
| Debugger | 断点调试 |
| Profiler | CPU、内存、能耗分析 |
| DevEco Studio Analyzer | 构建、包体积、性能分析 |
| 真机调试 | 验证权限、性能、系统能力 |

建议：

- Preview 只验证 UI，不代表真机表现。
- 性能问题必须真机验证。
- Release 构建和 Debug 构建表现不同。
- 日志、断点、Profiler 组合使用。

## 14. 可实践示例：ArkUI 状态拆分和组件边界

状态拆分的目标：父组件只保存页面级状态，子组件只接收必要字段，避免一个状态变化导致大范围刷新。

错误倾向：

```ts
// 页面把所有字段都塞进一个巨大对象
@State pageState: Record<string, Object> = {}
```

更推荐的结构：

```ts
interface User {
  id: number
  name: string
  age: number
}

interface UserPageState {
  loading: boolean
  users: User[]
  errorMessage: string
}
```

页面：

```ts
@Entry
@Component
struct UserPage {
  @State state: UserPageState = {
    loading: false,
    users: [],
    errorMessage: ''
  }

  async aboutToAppear(): Promise<void> {
    await this.loadUsers()
  }

  private async loadUsers(): Promise<void> {
    this.state = {
      ...this.state,
      loading: true,
      errorMessage: ''
    }

    try {
      const users = await this.requestUsers()

      this.state = {
        loading: false,
        users,
        errorMessage: ''
      }
    } catch (e) {
      this.state = {
        loading: false,
        users: [],
        errorMessage: '加载失败'
      }
    }
  }

  private async requestUsers(): Promise<User[]> {
    return [
      { id: 1, name: 'Tom', age: 18 },
      { id: 2, name: 'Jerry', age: 20 }
    ]
  }

  build() {
    Column() {
      if (this.state.loading) {
        LoadingProgress()
      } else if (this.state.errorMessage.length > 0) {
        ErrorView({
          message: this.state.errorMessage,
          onRetry: async () => {
            await this.loadUsers()
          }
        })
      } else {
        UserList({ users: this.state.users })
      }
    }
  }
}
```

错误组件：

```ts
@Component
struct ErrorView {
  @Prop message: string
  private onRetry: () => Promise<void> = async () => {}

  build() {
    Column({ space: 12 }) {
      Text(this.message)
        .fontColor(Color.Red)

      Button('重试')
        .onClick(() => {
          this.onRetry()
        })
    }
    .padding(16)
  }
}
```

列表组件：

```ts
@Component
struct UserList {
  @Prop users: User[]

  build() {
    List() {
      ForEach(this.users, (user: User) => {
        ListItem() {
          UserRow({ user })
        }
      }, (user: User) => user.id.toString())
    }
  }
}
```

要点：

- 页面状态是 `UserPageState`，不是散落的多个无关字段。
- `ErrorView` 和 `UserList` 都是纯展示组件。
- `ForEach` 提供稳定 key。
- 重试事件从子组件回调给页面。

## 15. 可实践示例：分页列表和加载状态

真实业务列表必须考虑分页、加载中、防重复请求和错误重试。

```ts
interface FeedItem {
  id: number
  title: string
}

@Entry
@Component
struct FeedPage {
  @State items: FeedItem[] = []
  @State page: number = 1
  @State loading: boolean = false
  @State noMore: boolean = false
  @State errorMessage: string = ''

  async aboutToAppear(): Promise<void> {
    await this.loadMore()
  }

  private async loadMore(): Promise<void> {
    if (this.loading || this.noMore) {
      return
    }

    this.loading = true
    this.errorMessage = ''

    try {
      const newItems = await this.requestPage(this.page)

      if (newItems.length === 0) {
        this.noMore = true
      } else {
        this.items = this.items.concat(newItems)
        this.page++
      }
    } catch (e) {
      this.errorMessage = '加载失败，请稍后重试'
    } finally {
      this.loading = false
    }
  }

  private async requestPage(page: number): Promise<FeedItem[]> {
    // 示例数据。真实项目这里调用网络层。
    if (page > 3) {
      return []
    }

    return [
      { id: page * 10 + 1, title: `第 ${page} 页数据 1` },
      { id: page * 10 + 2, title: `第 ${page} 页数据 2` }
    ]
  }

  build() {
    Column() {
      List() {
        ForEach(this.items, (item: FeedItem) => {
          ListItem() {
            Text(item.title)
              .fontSize(16)
              .padding(16)
          }
        }, (item: FeedItem) => item.id.toString())

        ListItem() {
          if (this.loading) {
            Text('加载中...')
              .padding(16)
          } else if (this.errorMessage.length > 0) {
            Button('重试')
              .onClick(() => {
                this.loadMore()
              })
          } else if (this.noMore) {
            Text('没有更多了')
              .fontColor(Color.Gray)
              .padding(16)
          }
        }
      }
      .onReachEnd(() => {
        this.loadMore()
      })
    }
  }
}
```

实践要点：

- `loading` 防止并发重复请求。
- `noMore` 避免到底后继续请求。
- 错误状态保留原列表，只提示底部重试。
- key 使用业务 id，不使用数组下标。

## 16. 可实践示例：过期异步任务保护

页面快速进入又离开时，异步请求可能在页面离开后才返回。可以通过请求序号忽略过期结果。

```ts
@Entry
@Component
struct SafeAsyncPage {
  @State text: string = ''
  private requestId: number = 0
  private active: boolean = false

  aboutToAppear(): void {
    this.active = true
    this.loadData()
  }

  aboutToDisappear(): void {
    // 页面不可见后，后续请求结果不再更新 UI
    this.active = false
  }

  private async loadData(): Promise<void> {
    const currentRequestId = ++this.requestId

    try {
      const result = await this.requestSlowData()

      if (!this.active) {
        return
      }

      if (currentRequestId !== this.requestId) {
        // 已经有更新的请求发出，当前结果过期
        return
      }

      this.text = result
    } catch (e) {
      if (this.active) {
        this.text = '加载失败'
      }
    }
  }

  private async requestSlowData(): Promise<string> {
    return 'data'
  }

  build() {
    Text(this.text)
  }
}
```

适用场景：

- 搜索框连续输入。
- 页面快速切换。
- 列表筛选条件变化。
- 网络返回顺序不稳定。

## 17. 可实践示例：Preferences 封装

页面不应该直接操作底层存储 API。建议封装 Store，让调用方只关心业务字段。

```ts
export class SettingsStore {
  private darkMode: boolean = false

  async setDarkMode(enabled: boolean): Promise<void> {
    // 真实项目中这里写入 Preferences
    this.darkMode = enabled
  }

  async getDarkMode(): Promise<boolean> {
    // 真实项目中这里从 Preferences 读取
    return this.darkMode
  }
}
```

页面使用：

```ts
@Entry
@Component
struct SettingsPage {
  @State darkMode: boolean = false
  private store: SettingsStore = new SettingsStore()

  async aboutToAppear(): Promise<void> {
    this.darkMode = await this.store.getDarkMode()
  }

  private async onDarkModeChanged(value: boolean): Promise<void> {
    this.darkMode = value
    await this.store.setDarkMode(value)
  }

  build() {
    Row() {
      Text('深色模式')
      Toggle({ type: ToggleType.Switch, isOn: this.darkMode })
        .onChange((value: boolean) => {
          this.onDarkModeChanged(value)
        })
    }
    .width('100%')
    .padding(16)
    .justifyContent(FlexAlign.SpaceBetween)
  }
}
```

封装收益：

- UI 不依赖具体存储 API。
- 后续从 Preferences 换成数据库时页面不变。
- 可以集中处理默认值、迁移和异常。

## 18. 可实践示例：权限申请流程建模

权限不要散落在页面事件里，建议抽成清晰流程：

```text
检查权限
  |
  +-- 已授权 -> 执行业务
  |
  +-- 未授权 -> 请求权限
        |
        +-- 同意 -> 执行业务
        +-- 拒绝 -> 展示降级提示
```

`module.json5` 声明：

```json5
{
  "module": {
    "requestPermissions": [
      {
        "name": "ohos.permission.CAMERA",
        "reason": "$string:camera_reason",
        "usedScene": {
          "abilities": ["EntryAbility"],
          "when": "inuse"
        }
      }
    ]
  }
}
```

页面侧伪代码：

```ts
@Entry
@Component
struct CameraEntryPage {
  @State message: string = ''

  private async openCamera(): Promise<void> {
    const granted = await this.ensureCameraPermission()

    if (!granted) {
      this.message = '没有相机权限，无法拍照'
      return
    }

    this.message = '可以打开相机'
    // 这里继续调用 Camera Kit 或跳转拍照页
  }

  private async ensureCameraPermission(): Promise<boolean> {
    // 真实项目中这里调用权限检查和请求 API
    // 返回 true 表示已授权，false 表示用户拒绝或系统不允许
    return true
  }

  build() {
    Column({ space: 12 }) {
      Button('打开相机')
        .onClick(() => {
          this.openCamera()
        })

      Text(this.message)
    }
    .padding(16)
  }
}
```

要点：

- 权限声明和运行时请求都要有。
- 权限拒绝不是异常情况，要有 UI 降级。
- `reason` 文案要说明真实用途。

## 19. 可实践示例：播放器生命周期封装

媒体对象要跟随页面生命周期释放。下面用简化 `PlayerController` 表达管理方式。

```ts
class PlayerController {
  private prepared: boolean = false
  private playing: boolean = false

  async prepare(url: string): Promise<void> {
    // 真实项目中这里创建播放器、设置数据源、监听事件
    console.info(`prepare url=${url}`)
    this.prepared = true
  }

  play(): void {
    if (!this.prepared) {
      return
    }

    this.playing = true
    console.info('play')
  }

  pause(): void {
    if (!this.playing) {
      return
    }

    this.playing = false
    console.info('pause')
  }

  release(): void {
    // 真实项目中这里释放播放器、取消监听、释放 Surface
    this.playing = false
    this.prepared = false
    console.info('release')
  }
}
```

页面：

```ts
@Entry
@Component
struct VideoPage {
  @State playing: boolean = false
  private controller: PlayerController = new PlayerController()

  async aboutToAppear(): Promise<void> {
    await this.controller.prepare('https://example.com/video.mp4')
  }

  aboutToDisappear(): void {
    // 页面离开必须释放，避免后台占用音频、解码器或 Surface
    this.controller.release()
  }

  private toggle(): void {
    if (this.playing) {
      this.controller.pause()
      this.playing = false
    } else {
      this.controller.play()
      this.playing = true
    }
  }

  build() {
    Column({ space: 12 }) {
      Text(this.playing ? '播放中' : '已暂停')

      Button(this.playing ? '暂停' : '播放')
        .onClick(() => {
          this.toggle()
        })
    }
    .padding(16)
  }
}
```

排查清单：

- 页面退出后仍播放：检查 `release()`。
- 首帧慢：检查网络、缓存、播放器 prepare 时机。
- 无声音：检查音量、音频焦点、后台策略。
- 相机/录音失败：检查权限声明和用户授权。

## 20. 总结

HarmonyOS 进阶知识可以按这条线理解：

```text
ArkUI 状态驱动渲染
  |
  v
Ability 生命周期和异步任务
  |
  v
ArkTS / HAR / HSP 模块化
  |
  v
启动、列表、内存、包体积优化
  |
  v
沙箱、权限、签名、数据和网络安全
  |
  v
相机、音频、视频、媒体生命周期
```

和 iOS、Android 对照学习时，重点是建立同构概念：页面生命周期、UI 渲染、状态刷新、数据隔离、权限授权、媒体资源释放和性能工具链。

## 21. 参考资料

- HarmonyOS 应用开发概述：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/application-dev-overview
- ArkUI 概述：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkui-overview
- ArkUI 状态管理：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-state-management-overview
- HarmonyOS 安全与权限：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/accesstoken-overview
- HarmonyOS 媒体开发：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/media-kit-intro
