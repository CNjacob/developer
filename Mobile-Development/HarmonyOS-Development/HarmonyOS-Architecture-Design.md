# HarmonyOS 常见架构设计、目录结构与示例代码

## 1. 为什么需要架构

HarmonyOS 初学项目通常从一个 `Index.ets` 开始，但业务变复杂后会出现：

- 页面里同时写 UI、网络、存储、权限和业务判断。
- 组件状态混乱，刷新逻辑难排查。
- 多个页面复制同样的数据请求代码。
- 项目目录没有规范，协作困难。
- 公共组件无法复用。

架构的目标是让代码职责清晰、依赖方向稳定、便于复用和测试。

## 2. 推荐基础分层

```text
UI Layer
  |
  v
State / ViewModel Layer
  |
  v
Domain Layer，可选
  |
  v
Data Layer
```

| 层 | 职责 |
|----|------|
| UI Layer | ArkUI 页面和组件，展示状态、接收用户操作 |
| State / ViewModel Layer | 页面状态、事件处理、调用业务或数据层 |
| Domain Layer | 可复用业务规则，可选 |
| Data Layer | 网络、本地存储、Repository、数据模型 |

## 3. 架构选型速查

| 架构 | 适合场景 | 新手建议 |
|------|----------|----------|
| 页面直写 | Demo、学习 API | 只适合入门 |
| 简单分层 | 小项目、业务简单 | 推荐起步 |
| MVVM | 大多数中小型业务 | 重点学习 |
| MVI / 单向数据流 | 状态复杂、事件多 | 有基础后学习 |
| Clean Architecture | 复杂业务、强测试要求 | 中大型项目使用 |
| 组件化 / 模块化 | 多业务线、多团队复用 | 项目变大后引入 |

## 4. 示例业务

后续示例都围绕用户列表：

```text
页面打开
  |
  v
加载用户列表
  |
  v
显示 loading
  |
  v
成功显示列表 / 失败显示错误
```

基础模型：

```ts
export interface User {
  id: number
  name: string
  age: number
}
```

## 5. 页面直写

### 5.1 目录结构

```text
entry/src/main/ets/
  pages/
    UserPage.ets
```

### 5.2 示例代码

```ts
interface User {
  id: number
  name: string
  age: number
}

@Entry
@Component
struct UserPage {
  @State loading: boolean = false
  @State users: User[] = []
  @State errorMessage: string = ''

  async aboutToAppear(): Promise<void> {
    this.loading = true
    this.errorMessage = ''

    try {
      // 示例：真实项目不要把请求逻辑直接写在页面里
      this.users = [
        { id: 1, name: 'Tom', age: 18 },
        { id: 2, name: 'Jerry', age: 20 }
      ]
    } catch (e) {
      this.errorMessage = '加载失败'
    } finally {
      this.loading = false
    }
  }

  build() {
    Column() {
      if (this.loading) {
        LoadingProgress()
      } else if (this.errorMessage.length > 0) {
        Text(this.errorMessage).fontColor(Color.Red)
      } else {
        List() {
          ForEach(this.users, (user: User) => {
            ListItem() {
              Text(`${user.name}, ${user.age}`)
                .padding(16)
            }
          }, (user: User) => user.id.toString())
        }
      }
    }
    .width('100%')
    .height('100%')
  }
}
```

### 5.3 优缺点

优点：

- 上手快。
- 文件少。

缺点：

- 页面容易膨胀。
- 数据逻辑无法复用。
- 难测试。

适合：Demo 和学习 API。

## 6. 简单分层架构

### 6.1 目录结构

```text
entry/src/main/ets/
  data/
    model/
      User.ets
    repository/
      UserRepository.ets
  pages/
    user/
      UserPage.ets
  components/
    UserCard.ets
```

### 6.2 Model

`data/model/User.ets`：

```ts
export interface User {
  id: number
  name: string
  age: number
}
```

### 6.3 Repository

`data/repository/UserRepository.ets`：

```ts
import { User } from '../model/User'

export class UserRepository {
  async getUsers(): Promise<User[]> {
    // 真实项目这里可以请求网络或读取本地数据库
    return [
      { id: 1, name: 'Tom', age: 18 },
      { id: 2, name: 'Jerry', age: 20 }
    ]
  }
}
```

### 6.4 Component

`components/UserCard.ets`：

```ts
import { User } from '../data/model/User'

@Component
export struct UserCard {
  @Prop user: User

  build() {
    Column() {
      Text(this.user.name)
        .fontSize(18)
        .fontWeight(FontWeight.Bold)

      Text(`Age: ${this.user.age}`)
        .fontSize(14)
        .fontColor(Color.Gray)
    }
    .alignItems(HorizontalAlign.Start)
    .padding(16)
    .width('100%')
  }
}
```

### 6.5 Page

`pages/user/UserPage.ets`：

```ts
import { User } from '../../data/model/User'
import { UserRepository } from '../../data/repository/UserRepository'
import { UserCard } from '../../components/UserCard'

@Entry
@Component
struct UserPage {
  @State loading: boolean = false
  @State users: User[] = []
  @State errorMessage: string = ''

  private repository: UserRepository = new UserRepository()

  async aboutToAppear(): Promise<void> {
    this.loading = true
    this.errorMessage = ''

    try {
      this.users = await this.repository.getUsers()
    } catch (e) {
      this.errorMessage = '加载失败'
    } finally {
      this.loading = false
    }
  }

  build() {
    Column() {
      if (this.loading) {
        LoadingProgress()
      } else if (this.errorMessage.length > 0) {
        Text(this.errorMessage).fontColor(Color.Red)
      } else {
        List() {
          ForEach(this.users, (user: User) => {
            ListItem() {
              UserCard({ user })
            }
          }, (user: User) => user.id.toString())
        }
      }
    }
  }
}
```

## 7. MVVM 架构

HarmonyOS 中可以借鉴 MVVM 思路：

```text
Page / Component
  |
  v
ViewModel
  |
  v
Repository
  |
  v
Api / Store
```

### 7.1 目录结构

```text
entry/src/main/ets/
  feature/
    user/
      data/
        UserApi.ets
        UserRepository.ets
        UserModel.ets
      ui/
        UserPage.ets
        UserViewModel.ets
        UserUiState.ets
        UserCard.ets
```

### 7.2 UiState

`UserUiState.ets`：

```ts
import { User } from '../data/UserModel'

export interface UserUiState {
  loading: boolean
  users: User[]
  errorMessage: string
}

export function createInitialUserUiState(): UserUiState {
  return {
    loading: false,
    users: [],
    errorMessage: ''
  }
}
```

### 7.3 ViewModel

`UserViewModel.ets`：

```ts
import { UserRepository } from '../data/UserRepository'
import { UserUiState, createInitialUserUiState } from './UserUiState'

export class UserViewModel {
  private repository: UserRepository
  state: UserUiState = createInitialUserUiState()

  constructor(repository: UserRepository) {
    this.repository = repository
  }

  async loadUsers(): Promise<UserUiState> {
    this.state = {
      ...this.state,
      loading: true,
      errorMessage: ''
    }

    try {
      const users = await this.repository.getUsers()

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

    // 返回新状态，让页面赋值给 @State 触发刷新
    return this.state
  }
}
```

### 7.4 Page

```ts
import { UserRepository } from '../data/UserRepository'
import { UserViewModel } from './UserViewModel'
import { UserUiState, createInitialUserUiState } from './UserUiState'

@Entry
@Component
struct UserPage {
  @State uiState: UserUiState = createInitialUserUiState()

  private viewModel: UserViewModel = new UserViewModel(
    new UserRepository()
  )

  async aboutToAppear(): Promise<void> {
    this.uiState = await this.viewModel.loadUsers()
  }

  build() {
    Column() {
      if (this.uiState.loading) {
        LoadingProgress()
      } else if (this.uiState.errorMessage.length > 0) {
        Text(this.uiState.errorMessage)
      } else {
        List() {
          ForEach(this.uiState.users, (user) => {
            ListItem() {
              Text(user.name)
            }
          }, (user) => user.id.toString())
        }
      }
    }
  }
}
```

说明：

- 页面只持有 `@State uiState`。
- ViewModel 负责加载和转换状态。
- Repository 负责数据来源。
- 业务复杂后可继续拆 UseCase。

## 8. MVI / 单向数据流

MVI 更强调事件和状态流向：

```text
UserAction
  |
  v
ViewModel.reduce
  |
  v
UserUiState
  |
  v
Page render
```

### 8.1 目录结构

```text
feature/user/
  data/
    UserRepository.ets
  mvi/
    UserAction.ets
    UserEffect.ets
    UserUiState.ets
    UserViewModel.ets
    UserPage.ets
```

### 8.2 Action

`UserAction.ets`：

```ts
export type UserAction =
  | { type: 'load' }
  | { type: 'retry' }
  | { type: 'userClick'; userId: number }
```

### 8.3 State

```ts
import { User } from '../data/UserModel'

export interface UserUiState {
  loading: boolean
  users: User[]
  errorMessage: string
}
```

### 8.4 ViewModel

```ts
export class UserViewModel {
  private repository: UserRepository
  state: UserUiState = {
    loading: false,
    users: [],
    errorMessage: ''
  }

  constructor(repository: UserRepository) {
    this.repository = repository
  }

  async dispatch(action: UserAction): Promise<UserUiState> {
    switch (action.type) {
      case 'load':
      case 'retry':
        return await this.loadUsers()
      case 'userClick':
        // 点击用户的导航行为可以交给 Page 层处理
        return this.state
    }
  }

  private async loadUsers(): Promise<UserUiState> {
    this.reduce({
      loading: true,
      users: this.state.users,
      errorMessage: ''
    })

    try {
      const users = await this.repository.getUsers()
      this.reduce({
        loading: false,
        users,
        errorMessage: ''
      })
    } catch (e) {
      this.reduce({
        loading: false,
        users: [],
        errorMessage: '加载失败'
      })
    }

    return this.state
  }

  private reduce(newState: UserUiState): void {
    this.state = newState
  }
}
```

### 8.5 Page

```ts
@Entry
@Component
struct UserPage {
  @State uiState: UserUiState = {
    loading: false,
    users: [],
    errorMessage: ''
  }

  private viewModel: UserViewModel = new UserViewModel(
    new UserRepository()
  )

  async aboutToAppear(): Promise<void> {
    this.uiState = await this.viewModel.dispatch({ type: 'load' })
  }

  async retry(): Promise<void> {
    this.uiState = await this.viewModel.dispatch({ type: 'retry' })
  }

  build() {
    Column() {
      if (this.uiState.loading) {
        LoadingProgress()
      } else if (this.uiState.errorMessage.length > 0) {
        Column() {
          Text(this.uiState.errorMessage)
          Button('重试')
            .onClick(() => {
              this.retry()
            })
        }
      } else {
        List() {
          ForEach(this.uiState.users, (user) => {
            ListItem() {
              Text(user.name)
                .onClick(() => {
                  // 用户意图统一转换为 action
                  this.viewModel.dispatch({
                    type: 'userClick',
                    userId: user.id
                  })
                })
            }
          }, (user) => user.id.toString())
        }
      }
    }
  }
}
```

适合：

- 页面状态复杂。
- 多个事件会影响同一份状态。
- 需要明确记录用户行为和状态变化。

## 9. Clean Architecture

### 9.1 目录结构

```text
feature/user/
  presentation/
    UserPage.ets
    UserViewModel.ets
    UserUiState.ets
  domain/
    model/
      User.ets
    repository/
      UserRepository.ets
    usecase/
      GetUsersUseCase.ets
  data/
    api/
      UserApi.ets
      UserDto.ets
    repository/
      UserRepositoryImpl.ets
```

### 9.2 Domain Model

```ts
export interface User {
  id: number
  name: string
  age: number
}
```

### 9.3 Repository 接口

```ts
import { User } from '../model/User'

export interface UserRepository {
  getUsers(): Promise<User[]>
}
```

### 9.4 UseCase

```ts
import { User } from '../model/User'
import { UserRepository } from '../repository/UserRepository'

export class GetUsersUseCase {
  private repository: UserRepository

  constructor(repository: UserRepository) {
    this.repository = repository
  }

  async execute(): Promise<User[]> {
    const users = await this.repository.getUsers()

    // 这里放业务规则，而不是 UI 展示逻辑
    return users.filter((user) => user.name.length > 0)
  }
}
```

### 9.5 DTO 与转换

```ts
export interface UserDto {
  id: number
  name?: string
  age?: number
}
```

```ts
import { User } from '../../domain/model/User'

export function toUser(dto: UserDto): User {
  return {
    id: dto.id,
    name: dto.name ?? 'Unknown',
    age: dto.age ?? 0
  }
}
```

### 9.6 Repository 实现

```ts
export class UserRepositoryImpl implements UserRepository {
  private api: UserApi

  constructor(api: UserApi) {
    this.api = api
  }

  async getUsers(): Promise<User[]> {
    const dtoList = await this.api.getUsers()

    // Data 层负责把接口模型转换为业务模型
    return dtoList.map((dto) => toUser(dto))
  }
}
```

### 9.7 Presentation

```ts
export class UserViewModel {
  private getUsersUseCase: GetUsersUseCase

  constructor(getUsersUseCase: GetUsersUseCase) {
    this.getUsersUseCase = getUsersUseCase
  }

  async loadUsers(): Promise<UserUiState> {
    try {
      const users = await this.getUsersUseCase.execute()
      return {
        loading: false,
        users,
        errorMessage: ''
      }
    } catch (e) {
      return {
        loading: false,
        users: [],
        errorMessage: '加载失败'
      }
    }
  }
}
```

## 10. 组件化与模块化

HarmonyOS 工程中常见复用方式：

| 类型 | 适合内容 |
|------|----------|
| entry HAP | 主应用入口和业务页面 |
| feature HAP | 独立业务能力 |
| HAR | 静态公共库，例如工具、组件、业务基础能力 |
| HSP | 动态共享包，例如多模块共享的运行时能力 |

### 10.1 目录结构示例

```text
HarmonyApp/
  entry/
  feature_user/
  feature_order/
  common_ui/
  common_network/
  common_utils/
```

职责：

| 模块 | 职责 |
|------|------|
| `entry` | 应用入口、主路由、组装能力 |
| `feature_user` | 用户业务 |
| `feature_order` | 订单业务 |
| `common_ui` | 通用组件和主题 |
| `common_network` | 网络封装 |
| `common_utils` | 工具函数 |

### 10.2 公共 UI 组件

`common_ui/src/main/ets/components/PrimaryButton.ets`：

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

业务模块使用：

```ts
PrimaryButton({
  text: '提交',
  onTap: () => {
    console.info('submit')
  }
})
```

## 11. 依赖方向

推荐依赖方向：

```text
pages/ui -> viewmodel -> usecase -> repository interface
data repository impl -> repository interface
```

不要反过来：

```text
data -> ui
domain -> ui
common -> feature
```

判断规则：

- 公共模块不能依赖业务模块。
- 数据层不能依赖页面。
- 业务规则不要依赖具体 UI。
- 能放 feature 内的不要过早放 common。

## 12. 数据模型划分

```text
DTO -> Entity/Storage Model -> Domain Model -> UI Model
```

| 类型 | 用途 |
|------|------|
| DTO | 接口返回 |
| Entity | 本地存储 |
| Domain Model | 业务规则 |
| UI Model | 页面展示 |

示例：

```ts
export interface UserDto {
  id: number
  nick_name?: string
}

export interface User {
  id: number
  name: string
}

export interface UserUiModel {
  title: string
}

export function dtoToDomain(dto: UserDto): User {
  return {
    id: dto.id,
    name: dto.nick_name ?? 'Unknown'
  }
}

export function domainToUi(user: User): UserUiModel {
  return {
    title: user.name
  }
}
```

## 13. 新手推荐模板

小项目推荐：

```text
entry/src/main/ets/
  common/
  components/
  feature/
    user/
      data/
      ui/
```

中型项目推荐：

```text
entry/src/main/ets/
  core/
  feature/
    user/
      data/
      domain/
      presentation/
```

大型项目推荐：

```text
entry/
feature_user/
feature_order/
common_ui/
common_network/
common_utils/
```

## 14. 常见反模式

| 反模式 | 问题 |
|--------|------|
| 所有代码写在 `Index.ets` | 页面臃肿，无法复用 |
| 页面直接调用底层存储和网络 | 难测试，难替换 |
| common 模块放业务代码 | 公共模块被污染 |
| DTO 直接用于 UI | 接口变化影响页面 |
| 每个函数都建 UseCase | 样板代码过多 |
| 过早模块化 | 构建和依赖管理复杂度上升 |

## 15. 架构演进建议

```text
页面直写
  |
  v
简单分层：Page + Component + Repository
  |
  v
MVVM：UiState + ViewModel + Repository
  |
  v
Clean：UseCase + Domain Model + Repository Interface
  |
  v
模块化：feature + common HAR/HSP
```

不要为了架构而架构。每次升级都应该解决真实问题：

- 重复代码多。
- 页面太大。
- 业务规则难复用。
- 测试困难。
- 多人协作冲突多。
- 构建变慢。

## 16. 参考资料

- HarmonyOS 应用架构概述：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/application-dev-overview
- Stage 模型包结构：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/application-package-structure-stage
- HAR 开发：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/har-package
- HSP 开发：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/in-app-hsp
- ArkUI 状态管理：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-state-management-overview

## 17. 总结

HarmonyOS 架构设计重点：

| 主题 | 建议 |
|------|------|
| 小项目 | 简单分层即可 |
| 常规业务 | MVVM + Repository |
| 复杂状态 | MVI / 单向数据流 |
| 复杂业务 | Clean Architecture |
| 多团队复用 | HAR / HSP / 模块化 |

新手最推荐先掌握：

```text
Page 只负责 UI
Component 负责复用 UI
ViewModel 负责状态和事件
Repository 负责数据
UseCase 只在业务规则复杂时引入
```
