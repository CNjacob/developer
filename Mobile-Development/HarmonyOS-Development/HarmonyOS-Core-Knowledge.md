# HarmonyOS 基础知识与常用开发技术

## 1. HarmonyOS 应用开发核心概念

现代 HarmonyOS 应用开发常见关键词：

| 概念 | 说明 |
|------|------|
| Stage 模型 | 当前主流应用模型，按 AbilityStage、UIAbility、ExtensionAbility 组织 |
| UIAbility | 带 UI 的应用组件，类似页面入口和生命周期容器 |
| Page | ArkUI 页面文件，通常位于 `pages/` 目录 |
| ArkTS | HarmonyOS 应用主要开发语言 |
| ArkUI | 声明式 UI 框架 |
| HAP | 应用安装包基本单元 |
| HAR | 静态共享包 |
| HSP | 动态共享包 |
| Want | 组件间跳转和参数传递的数据结构 |

## 2. Stage 模型

Stage 模型把应用分为应用级、模块级、Ability 级：

```text
Application
  |
  v
AbilityStage
  |
  v
UIAbility / ExtensionAbility
  |
  v
Page / Service / Widget 等能力
```

常见文件：

```text
entry/src/main/ets/
  entryability/
    EntryAbility.ets
  pages/
    Index.ets
```

## 3. UIAbility 生命周期

UIAbility 是带界面的 Ability，常见生命周期：

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
    // Ability 创建时调用，适合做轻量初始化
    console.info('EntryAbility onCreate')
  }

  onWindowStageCreate(windowStage: window.WindowStage): void {
    // 窗口创建后加载首页
    windowStage.loadContent('pages/Index', (err) => {
      if (err.code) {
        console.error(`load page failed: ${err.message}`)
        return
      }

      console.info('load page success')
    })
  }

  onForeground(): void {
    // 应用进入前台
    console.info('EntryAbility onForeground')
  }

  onBackground(): void {
    // 应用进入后台，适合暂停动画、保存状态
    console.info('EntryAbility onBackground')
  }

  onDestroy(): void {
    // Ability 销毁，释放资源
    console.info('EntryAbility onDestroy')
  }
}
```

## 4. ArkUI 声明式 UI

ArkUI 使用组件函数声明界面。

```ts
@Entry
@Component
struct Index {
  @State count: number = 0

  build() {
    Column() {
      Text(`Count: ${this.count}`)
        .fontSize(24)

      Button('Add')
        .margin({ top: 16 })
        .onClick(() => {
          this.count++
        })
    }
    .width('100%')
    .height('100%')
    .justifyContent(FlexAlign.Center)
  }
}
```

核心思想：

- UI 是状态的函数。
- 状态变化驱动 UI 刷新。
- 不要手动查找控件再修改控件属性，应修改状态。

## 5. 常用布局组件

### 5.1 Column

垂直布局：

```ts
Column({ space: 12 }) {
  Text('Title')
  Text('Subtitle')
  Button('Submit')
}
.padding(16)
```

### 5.2 Row

水平布局：

```ts
Row({ space: 8 }) {
  Text('Name')
  Text('Tom')
}
.width('100%')
.justifyContent(FlexAlign.SpaceBetween)
```

### 5.3 Stack

层叠布局：

```ts
Stack() {
  Image($r('app.media.background'))
    .width('100%')
    .height(200)

  Text('Overlay')
    .fontColor(Color.White)
}
```

### 5.4 List

列表：

```ts
@Entry
@Component
struct UserListPage {
  private users: string[] = ['Tom', 'Jerry', 'Alice']

  build() {
    List() {
      ForEach(this.users, (name: string) => {
        ListItem() {
          Text(name)
            .fontSize(18)
            .padding(16)
        }
      })
    }
  }
}
```

## 6. 状态管理基础

常见状态装饰器：

| 装饰器 | 作用 |
|--------|------|
| `@State` | 组件内部状态 |
| `@Prop` | 父组件向子组件传递只读数据 |
| `@Link` | 父子组件双向同步状态 |
| `@Provide` / `@Consume` | 跨层级状态提供和消费 |
| `@Observed` / `@ObjectLink` | 对象级状态观察 |

### 6.1 @State

```ts
@Component
struct Counter {
  @State count: number = 0

  build() {
    Button(`Count: ${this.count}`)
      .onClick(() => {
        this.count++
      })
  }
}
```

### 6.2 @Prop

```ts
@Component
struct UserCard {
  @Prop name: string

  build() {
    Text(this.name)
  }
}
```

父组件：

```ts
UserCard({ name: 'Tom' })
```

### 6.3 @Link

```ts
@Component
struct SwitchRow {
  @Link enabled: boolean

  build() {
    Row() {
      Text('Enabled')
      Toggle({ type: ToggleType.Switch, isOn: this.enabled })
        .onChange((value: boolean) => {
          this.enabled = value
        })
    }
  }
}
```

父组件：

```ts
@State enabled: boolean = false

SwitchRow({ enabled: $enabled })
```

## 7. 页面路由

常用页面跳转可以使用 router。

```ts
import router from '@ohos.router'

router.pushUrl({
  url: 'pages/Detail',
  params: {
    userId: 1001
  }
})
```

接收参数：

```ts
import router from '@ohos.router'

@Entry
@Component
struct Detail {
  @State userId: number = 0

  aboutToAppear(): void {
    const params = router.getParams() as Record<string, number>
    this.userId = params['userId'] ?? 0
  }

  build() {
    Text(`userId=${this.userId}`)
  }
}
```

返回：

```ts
router.back()
```

## 8. Want 与 Ability 跳转

Want 用于描述要启动的 Ability 及参数。

```ts
import common from '@ohos.app.ability.common'

let context = getContext(this) as common.UIAbilityContext

context.startAbility({
  bundleName: 'com.example.demo',
  abilityName: 'EntryAbility',
  parameters: {
    from: 'home'
  }
})
```

适合场景：

- 启动本应用其他 Ability。
- 跨应用拉起能力。
- 传递启动参数。

## 9. 网络请求

可使用系统网络能力或封装 HTTP 客户端。示例以伪代码方式展示分层思路：

```ts
export interface User {
  id: number
  name: string
}

export class UserApi {
  async getUsers(): Promise<User[]> {
    // 真实项目中这里调用 http API
    return [
      { id: 1, name: 'Tom' },
      { id: 2, name: 'Jerry' }
    ]
  }
}
```

页面调用：

```ts
@Entry
@Component
struct UserPage {
  @State users: User[] = []
  private api: UserApi = new UserApi()

  async aboutToAppear(): Promise<void> {
    try {
      this.users = await this.api.getUsers()
    } catch (e) {
      console.error(`load users failed: ${JSON.stringify(e)}`)
    }
  }

  build() {
    List() {
      ForEach(this.users, (user: User) => {
        ListItem() {
          Text(user.name)
        }
      })
    }
  }
}
```

注意：

- 网络请求要处理超时、失败、空数据。
- 页面层不要直接堆大量请求逻辑。
- 建议封装到 `data/api` 或 `repository`。

## 10. 本地存储

常见存储类型：

| 类型 | 适合 |
|------|------|
| Preferences | 少量键值配置 |
| 文件 | 文本、图片、缓存文件 |
| 数据库 | 结构化数据 |
| 安全存储 | Token、敏感数据 |

Preferences 伪代码结构：

```ts
export class SettingsStore {
  async saveDarkMode(enabled: boolean): Promise<void> {
    // 保存键值配置
  }

  async getDarkMode(): Promise<boolean> {
    // 读取键值配置
    return false
  }
}
```

建议：

- UI 层不要直接依赖底层存储 API。
- 用 Repository 或 Store 封装。
- 敏感数据不要明文存储。

## 11. 权限

权限使用流程：

```text
module.json5 声明权限
  |
  v
运行时检查权限
  |
  v
必要时请求授权
  |
  v
授权成功后调用系统能力
```

`module.json5` 示例：

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

建议：

- 用到时再申请。
- 申请前说明用途。
- 被拒绝后提供降级方案。
- 不要申请不需要的权限。

## 12. 资源与多语言

资源目录：

```text
resources/
  base/
    element/
      string.json
      color.json
    media/
```

字符串：

```json
{
  "string": [
    {
      "name": "login",
      "value": "登录"
    }
  ]
}
```

页面中引用：

```ts
Text($r('app.string.login'))
```

## 13. 组件化

组件示例：

```ts
@Component
export struct UserCard {
  @Prop name: string
  @Prop age: number

  build() {
    Column() {
      Text(this.name)
        .fontSize(18)
        .fontWeight(FontWeight.Bold)

      Text(`Age: ${this.age}`)
        .fontSize(14)
        .fontColor(Color.Gray)
    }
    .padding(16)
    .borderRadius(8)
    .backgroundColor('#FFFFFF')
  }
}
```

使用：

```ts
UserCard({
  name: 'Tom',
  age: 18
})
```

## 14. 常用工程目录建议

```text
entry/src/main/ets/
  common/
    constants/
    utils/
  data/
    api/
    model/
    repository/
  pages/
    home/
      HomePage.ets
    detail/
      DetailPage.ets
  components/
    UserCard.ets
  entryability/
    EntryAbility.ets
```

职责：

| 目录 | 作用 |
|------|------|
| `common` | 常量、工具、通用能力 |
| `data` | 数据模型、接口、仓库 |
| `pages` | 页面 |
| `components` | 可复用 UI 组件 |
| `entryability` | Ability 入口 |

## 15. 调试工具

常用调试方式：

| 工具 | 用途 |
|------|------|
| Previewer | 快速预览 UI |
| Console | 查看日志 |
| Debugger | 断点调试 |
| Profiler | 性能分析 |
| Device Manager | 管理设备 |

调试建议：

- UI 问题先用 Previewer。
- 生命周期问题用日志。
- 数据问题用断点。
- 性能问题用 Profiler。
- 真机问题不要只依赖预览器判断。

## 16. 新手常见误区

| 误区 | 正确理解 |
|------|----------|
| 普通字段变化会刷新 UI | 需要响应式状态装饰器 |
| 页面里直接写所有网络和存储 | 应逐步拆到 Repository / Store |
| Preview 正常就代表真机正常 | 真机权限、性能、系统 API 都要验证 |
| 所有内容都放 `Index.ets` | 页面、组件、数据层要拆分 |
| 资源硬编码在页面里 | 文案、颜色、图片应资源化 |
| 不关注签名 | 真机调试和发布都离不开签名 |

## 17. 学习路线

```text
ArkTS 基础
  |
  v
ArkUI 组件和状态
  |
  v
UIAbility 生命周期
  |
  v
页面路由和参数
  |
  v
网络、存储、权限
  |
  v
分层架构和组件化
  |
  v
调试、性能、发布
```

## 18. 参考资料

- HarmonyOS 应用开发概述：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/application-dev-overview
- Stage 模型包结构：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/application-package-structure-stage
- UIAbility：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/uiability-overview
- ArkUI 概述：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkui-overview

## 19. 总结

HarmonyOS 基础开发要抓住四条线：

| 主线 | 内容 |
|------|------|
| 应用模型 | Stage、UIAbility、Page、Want |
| UI 开发 | ArkUI、组件、状态、路由 |
| 系统能力 | 权限、网络、存储、设备能力 |
| 工程治理 | 目录结构、组件化、调试、签名发布 |

初学者最好的练习顺序是：一个页面、一个组件、一次状态更新、一次页面跳转、一次数据加载、一次真机运行。
