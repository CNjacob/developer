# HarmonyOS 环境搭建

## 1. 学习目标

这篇文档面向鸿蒙应用开发初学者，目标是把开发环境从零搭起来，并理解每个工具的作用。

完成后你应该能做到：

- 安装 DevEco Studio。
- 创建 HarmonyOS 应用项目。
- 理解 Stage 模型项目结构。
- 配置 HarmonyOS SDK。
- 使用预览器、模拟器或真机运行页面。
- 理解 HAP、HAR、HSP 的基本区别。
- 完成调试、日志查看、签名和构建。

## 2. 工具链总览

```text
DevEco Studio
    |
    |-- HarmonyOS SDK: API、编译工具、预览、模拟器/设备工具
    |-- ArkTS: 主要开发语言
    |-- ArkUI: 声明式 UI 框架
    |-- Hvigor: 工程构建工具
    |-- HDC: 设备连接和调试工具
    |-- AppGallery Connect: 证书、签名、发布
```

| 工具 | 作用 |
|------|------|
| DevEco Studio | 官方 IDE，创建、开发、调试、构建 HarmonyOS 应用 |
| HarmonyOS SDK | API、编译器、工具链、设备调试工具 |
| ArkTS | HarmonyOS 应用主要开发语言，基于 TypeScript 扩展 |
| ArkUI | 声明式 UI 框架 |
| Hvigor | 工程构建系统，类似 Android Gradle 的角色 |
| HDC | HarmonyOS Device Connector，用于连接设备、安装、日志、调试 |

## 3. 安装前准备

建议配置：

| 项目 | 建议 |
|------|------|
| 内存 | 16GB 及以上 |
| 磁盘 | 至少预留 20GB |
| 系统 | macOS / Windows，按 DevEco Studio 当前支持列表选择 |
| 网络 | SDK 下载和设备调试需要稳定网络 |

macOS 建议先准备：

```bash
# 基础命令行工具
xcode-select --install

# 可选：常用开发工具
brew install git tree jq
```

## 4. 安装 DevEco Studio

从华为开发者官网下载 DevEco Studio 并安装。首次启动通常需要完成：

```text
安装 IDE
  |
  v
下载 HarmonyOS SDK
  |
  v
配置 Node / Ohpm / Hvigor 等工具链
  |
  v
登录华为开发者账号，可选但真机调试和签名常用
```

初学阶段建议使用默认安装路径和默认 SDK 配置，减少环境变量问题。

## 5. SDK 管理

在 DevEco Studio 中进入：

```text
Settings / Preferences
  |
  v
SDK
```

常见 SDK 内容：

| 内容 | 作用 |
|------|------|
| ArkTS / ArkUI API | 应用开发 API |
| Toolchains | 编译、打包、签名工具 |
| Previewer | 页面预览 |
| Emulator / Device Tools | 模拟器和设备调试相关工具 |

项目开发时要注意：

- `compileSdkVersion` 或项目选择的 API 版本要和已安装 SDK 匹配。
- 团队项目不要随意升级 SDK，先确认构建和兼容性。
- 新 API 要确认目标设备系统版本是否支持。

## 6. 创建第一个项目

DevEco Studio 中选择：

```text
Create Project
  |
  v
Application
  |
  v
Empty Ability
```

初学建议：

| 配置 | 建议 |
|------|------|
| Language | ArkTS |
| Model | Stage |
| UI | ArkUI |
| Device | Phone / Tablet 按学习目标选择 |

创建完成后，先直接运行默认页面，确认环境闭环正常。

## 7. Stage 模型项目结构

典型结构：

```text
HarmonyApp/
  AppScope/
    app.json5
  entry/
    src/main/
      module.json5
      ets/
        entryability/
          EntryAbility.ets
        pages/
          Index.ets
      resources/
        base/
          element/
            string.json
            color.json
          media/
    build-profile.json5
    hvigorfile.ts
  oh-package.json5
  build-profile.json5
  hvigorfile.ts
```

| 文件/目录 | 作用 |
|-----------|------|
| `AppScope/app.json5` | 应用级配置 |
| `entry/` | 默认主模块 |
| `module.json5` | 模块配置，声明 Ability、页面、权限等 |
| `EntryAbility.ets` | UIAbility 入口 |
| `pages/Index.ets` | 页面文件 |
| `resources/` | 字符串、颜色、图片等资源 |
| `oh-package.json5` | 依赖配置 |
| `hvigorfile.ts` | 构建脚本 |

## 8. 第一个页面

`entry/src/main/ets/pages/Index.ets`：

```ts
@Entry
@Component
struct Index {
  @State message: string = 'Hello HarmonyOS'

  build() {
    Column() {
      Text(this.message)
        .fontSize(24)
        .fontWeight(FontWeight.Bold)

      Button('Click')
        .margin({ top: 16 })
        .onClick(() => {
          // 修改 @State 状态后，ArkUI 会自动刷新依赖该状态的 UI
          this.message = 'Clicked'
        })
    }
    .width('100%')
    .height('100%')
    .justifyContent(FlexAlign.Center)
  }
}
```

关键点：

- `@Entry` 表示页面入口组件。
- `@Component` 表示声明式 UI 组件。
- `@State` 是组件内部响应式状态。
- `build()` 描述 UI 结构。
- `Column()` 是垂直布局容器。

## 9. UIAbility 入口

`EntryAbility.ets` 通常负责窗口创建和页面加载。

简化示例：

```ts
import UIAbility from '@ohos.app.ability.UIAbility'
import window from '@ohos.window'

export default class EntryAbility extends UIAbility {
  onWindowStageCreate(windowStage: window.WindowStage): void {
    // 加载 pages/Index 页面
    windowStage.loadContent('pages/Index', (err) => {
      if (err.code) {
        console.error(`loadContent failed, code=${err.code}, message=${err.message}`)
        return
      }

      console.info('loadContent success')
    })
  }
}
```

## 10. 运行方式

常见运行方式：

| 方式 | 适用 |
|------|------|
| Previewer | 快速预览 ArkUI 页面 |
| Local Emulator | 本地模拟设备 |
| Remote Emulator | 云端或远程模拟设备，视账号和环境能力而定 |
| Real Device | 真机验证性能、权限、设备能力 |

建议顺序：

```text
Previewer 看 UI
  |
  v
模拟器跑流程
  |
  v
真机验证权限、性能、系统能力
```

## 11. 真机调试

真机调试通常需要：

- 开启开发者模式。
- 开启 USB 调试或相关调试能力。
- 电脑信任设备。
- DevEco Studio 能识别设备。
- 调试签名配置正确。

如果设备无法识别：

```text
检查数据线
  |
  v
检查设备授权弹窗
  |
  v
重启 IDE 或设备连接服务
  |
  v
确认 SDK / HDC 工具可用
```

## 12. HDC 常用概念

HDC 类似 Android 的 ADB，用于连接设备和执行调试命令。

常见用途：

```text
查看设备
安装应用
卸载应用
查看日志
推送文件
执行 shell 命令
```

实际命令会随 SDK 工具版本变化，初学阶段优先使用 DevEco Studio 图形化能力；需要自动化时再使用命令行。

## 13. 日志调试

ArkTS 中可以使用日志：

```ts
console.info('normal info')
console.warn('warning message')
console.error('error message')
```

带变量：

```ts
let userId: number = 1001
console.info(`load user, userId=${userId}`)
```

建议：

- 普通流程用 `info`。
- 可恢复问题用 `warn`。
- 异常和失败用 `error`。
- 不要输出用户隐私、Token、密码。

## 14. 资源文件

字符串资源示例：

`resources/base/element/string.json`：

```json
{
  "string": [
    {
      "name": "app_name",
      "value": "Harmony Demo"
    },
    {
      "name": "login",
      "value": "登录"
    }
  ]
}
```

使用资源可以：

- 支持多语言。
- 统一管理文案。
- 避免硬编码。

## 15. HAP / HAR / HSP

| 类型 | 说明 |
|------|------|
| HAP | Harmony Ability Package，应用安装和运行的基本包 |
| HAR | Harmony Archive，静态共享包，类似源码/静态库复用 |
| HSP | Harmony Shared Package，动态共享包，适合运行时共享能力 |

初学理解：

```text
普通 App 页面和业务：HAP
公共工具和 UI 组件：HAR
多模块动态共享能力：HSP
```

## 16. 签名与发布

开发和发布通常涉及：

| 项目 | 说明 |
|------|------|
| 证书 | 标识开发者身份 |
| Profile | 描述应用、设备、权限等授权信息 |
| 调试签名 | 本地调试使用 |
| 发布签名 | 上架发布使用 |

初学阶段：

- 本地运行可先使用 IDE 自动配置。
- 真机调试需要按账号和设备配置调试签名。
- 发布前必须梳理包名、证书、权限和隐私说明。

## 17. 常见问题

### 17.1 SDK 未安装或版本不匹配

现象：

```text
项目无法同步
API 找不到
构建失败
```

处理：

- 打开 SDK Manager。
- 安装项目要求的 SDK。
- 确认项目配置中的 API 版本。

### 17.2 Preview 能显示，真机运行失败

可能原因：

- 页面使用了真机不支持的系统 API。
- 权限未声明。
- 签名不正确。
- 设备系统版本低于目标 API。

### 17.3 页面不刷新

常见原因：

- 普通字段变化不会触发 UI 刷新。
- 应使用 `@State`、`@Prop`、`@Link`、`@Observed` 等状态装饰器。

### 17.4 资源找不到

检查：

- 文件是否放在正确的 `resources` 目录。
- JSON 格式是否正确。
- 资源名是否拼写一致。
- 构建缓存是否需要清理。

## 18. 最小验证清单

搭建完成后确认：

- DevEco Studio 能启动。
- SDK Manager 能看到已安装 SDK。
- 能创建 Empty Ability 项目。
- `Index.ets` 能预览。
- 能运行到模拟器或真机。
- 能看到 `console.info` 日志。
- 能构建 HAP。

## 19. 学习路径

```text
环境搭建
  |
  v
ArkTS 基础语法
  |
  v
ArkUI 页面和状态
  |
  v
UIAbility / Page / Router
  |
  v
网络 / 存储 / 权限
  |
  v
分层架构 / 组件化
  |
  v
调试 / 性能 / 签名发布
```

## 20. 参考资料

- HarmonyOS 应用开发文档：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/application-dev-overview
- ArkTS 入门：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-get-started
- Stage 模型包结构：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/application-package-structure-stage
- ArkUI 概述：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkui-overview

## 21. 总结

HarmonyOS 环境搭建的最小闭环是：

```text
DevEco Studio
  |
  v
HarmonyOS SDK
  |
  v
ArkTS + ArkUI 页面
  |
  v
Preview / Emulator / Real Device
  |
  v
HAP 构建和调试
```

初学阶段不要急着研究复杂工程。先跑通一个页面、一个按钮、一次状态更新、一次真机运行，再逐步学习 Ability、路由、数据和架构。
