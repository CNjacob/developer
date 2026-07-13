# HarmonyOS ArkTS 语法知识点

## 1. ArkTS 是什么

ArkTS 是 HarmonyOS 应用开发的主要语言，基于 TypeScript 语法扩展，并面向 ArkUI 声明式开发做了增强。

学习 ArkTS 时要同时掌握两部分：

| 部分 | 内容 |
|------|------|
| 语言基础 | 变量、类型、函数、类、接口、泛型、异步 |
| ArkUI 扩展 | `@Component`、`@State`、`build()`、组件链式调用 |

## 2. Hello World

普通函数：

```ts
function main(): void {
  console.info('Hello ArkTS')
}
```

ArkUI 页面：

```ts
@Entry
@Component
struct Index {
  build() {
    Text('Hello HarmonyOS')
      .fontSize(24)
  }
}
```

说明：

- `@Entry` 表示入口组件。
- `@Component` 表示 ArkUI 组件。
- `struct` 是声明组件结构的常见写法。
- `build()` 描述 UI。

## 3. 变量

```ts
let age: number = 18       // 可重新赋值
const name: string = 'Tom' // 常量，不可重新赋值

age = 19
// name = 'Jerry' // 编译错误
```

建议：

- 默认使用 `const`。
- 需要变化时使用 `let`。
- 不要滥用 `any`。

## 4. 基础类型

```ts
let id: number = 1001
let title: string = 'HarmonyOS'
let enabled: boolean = true
let emptyValue: null = null
let notSet: undefined = undefined
```

数组：

```ts
let names: string[] = ['Tom', 'Jerry']
let scores: Array<number> = [90, 80]
```

对象类型：

```ts
let user: { id: number; name: string } = {
  id: 1,
  name: 'Tom'
}
```

## 5. 类型推断

```ts
let count = 10       // 推断为 number
let message = 'Hi'   // 推断为 string
let visible = true   // 推断为 boolean
```

虽然可以推断，但公共 API 建议写明确类型：

```ts
function getUserName(id: number): string {
  return `user-${id}`
}
```

## 6. 联合类型

```ts
let value: string | number = '100'
value = 100
```

类型缩窄：

```ts
function printValue(value: string | number): void {
  if (typeof value === 'string') {
    console.info(`length=${value.length}`)
  } else {
    console.info(`number=${value}`)
  }
}
```

## 7. 函数

普通函数：

```ts
function add(a: number, b: number): number {
  return a + b
}
```

箭头函数：

```ts
const add = (a: number, b: number): number => {
  return a + b
}
```

默认参数：

```ts
function request(url: string, timeout: number = 10): void {
  console.info(`url=${url}, timeout=${timeout}`)
}
```

可选参数：

```ts
function greet(name?: string): string {
  return `Hello ${name ?? 'Guest'}`
}
```

## 8. 条件语句

```ts
let score = 85

if (score >= 90) {
  console.info('A')
} else if (score >= 60) {
  console.info('Pass')
} else {
  console.info('Fail')
}
```

`switch`：

```ts
let tab = 'home'

switch (tab) {
  case 'home':
    console.info('Home')
    break
  case 'profile':
    console.info('Profile')
    break
  default:
    console.info('Unknown')
}
```

## 9. 循环

```ts
for (let i = 0; i < 5; i++) {
  console.info(`${i}`)
}
```

遍历数组：

```ts
let names: string[] = ['Tom', 'Jerry']

for (const name of names) {
  console.info(name)
}
```

`forEach`：

```ts
names.forEach((name: string, index: number) => {
  console.info(`${index}: ${name}`)
})
```

## 10. 接口

接口用于定义对象形状。

```ts
interface User {
  id: number
  name: string
  age?: number
}
```

使用：

```ts
const user: User = {
  id: 1,
  name: 'Tom'
}
```

函数参数：

```ts
function renderUser(user: User): void {
  console.info(`name=${user.name}`)
}
```

## 11. type 类型别名

```ts
type UserId = number
type LoadState = 'idle' | 'loading' | 'success' | 'error'
```

使用：

```ts
let state: LoadState = 'loading'
```

建议：

- 对象形状可用 `interface`。
- 联合类型、函数类型、别名用 `type` 更自然。

## 12. 类

```ts
class UserRepository {
  private cache: User[] = []

  async getUsers(): Promise<User[]> {
    return this.cache
  }

  saveUsers(users: User[]): void {
    this.cache = users
  }
}
```

构造函数：

```ts
class UserService {
  private repository: UserRepository

  constructor(repository: UserRepository) {
    this.repository = repository
  }
}
```

## 13. 访问修饰符

| 修饰符 | 说明 |
|--------|------|
| `public` | 默认，外部可访问 |
| `private` | 类内部访问 |
| `protected` | 类内部和子类访问 |

示例：

```ts
class Counter {
  private count: number = 0

  public increase(): void {
    this.count++
  }

  public getCount(): number {
    return this.count
  }
}
```

## 14. 泛型

```ts
class ApiResult<T> {
  data: T
  message: string

  constructor(data: T, message: string) {
    this.data = data
    this.message = message
  }
}
```

使用：

```ts
const result = new ApiResult<User>(
  { id: 1, name: 'Tom' },
  'ok'
)
```

泛型函数：

```ts
function first<T>(items: T[]): T | undefined {
  return items.length > 0 ? items[0] : undefined
}
```

## 15. 空值处理

可选属性：

```ts
interface User {
  id: number
  name?: string
}
```

安全读取：

```ts
function getDisplayName(user: User): string {
  return user.name ?? 'Unknown'
}
```

可选链：

```ts
let length = user.name?.length ?? 0
```

建议：

- 不要假设接口字段一定存在。
- UI 展示前要做兜底。
- 少用强制断言。

## 16. 异步 Promise

```ts
function delay(ms: number): Promise<void> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve()
    }, ms)
  })
}
```

`async/await`：

```ts
async function loadData(): Promise<string> {
  await delay(500)
  return 'done'
}
```

错误处理：

```ts
async function loadUser(): Promise<void> {
  try {
    const data = await loadData()
    console.info(data)
  } catch (e) {
    console.error(`load failed: ${JSON.stringify(e)}`)
  }
}
```

## 17. ArkUI 组件

基本组件：

```ts
@Component
struct UserCard {
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

## 18. ArkUI 状态装饰器

### 18.1 @State

```ts
@State count: number = 0
```

组件内部状态，变化后刷新 UI。

### 18.2 @Prop

```ts
@Prop title: string
```

父组件传给子组件的只读数据。

### 18.3 @Link

```ts
@Link enabled: boolean
```

父子双向绑定。

### 18.4 示例

```ts
@Component
struct Parent {
  @State enabled: boolean = false

  build() {
    Column() {
      Text(`enabled=${this.enabled}`)

      Child({ enabled: $enabled })
    }
  }
}

@Component
struct Child {
  @Link enabled: boolean

  build() {
    Button('Toggle')
      .onClick(() => {
        this.enabled = !this.enabled
      })
  }
}
```

## 19. 生命周期函数

组件常用生命周期：

```ts
@Entry
@Component
struct Index {
  aboutToAppear(): void {
    // 组件即将出现，适合加载数据
    console.info('aboutToAppear')
  }

  aboutToDisappear(): void {
    // 组件即将消失，适合释放监听
    console.info('aboutToDisappear')
  }

  build() {
    Text('Lifecycle')
  }
}
```

## 20. 列表渲染

```ts
interface User {
  id: number
  name: string
}

@Entry
@Component
struct UserList {
  @State users: User[] = [
    { id: 1, name: 'Tom' },
    { id: 2, name: 'Jerry' }
  ]

  build() {
    List() {
      ForEach(this.users, (user: User) => {
        ListItem() {
          Text(user.name)
            .fontSize(18)
            .padding(16)
        }
      }, (user: User) => user.id.toString())
    }
  }
}
```

第三个参数是 key 生成函数，有助于列表更新稳定。

## 21. 条件渲染

```ts
@Entry
@Component
struct LoadPage {
  @State loading: boolean = true
  @State errorMessage: string = ''

  build() {
    Column() {
      if (this.loading) {
        LoadingProgress()
      } else if (this.errorMessage.length > 0) {
        Text(this.errorMessage)
          .fontColor(Color.Red)
      } else {
        Text('Content')
      }
    }
  }
}
```

## 22. 页面数据加载示例

```ts
interface User {
  id: number
  name: string
}

class UserRepository {
  async getUsers(): Promise<User[]> {
    // 模拟异步请求
    return [
      { id: 1, name: 'Tom' },
      { id: 2, name: 'Jerry' }
    ]
  }
}

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
        Text(this.errorMessage)
      } else {
        List() {
          ForEach(this.users, (user: User) => {
            ListItem() {
              Text(user.name)
            }
          }, (user: User) => user.id.toString())
        }
      }
    }
  }
}
```

## 23. 常见误区

| 误区 | 正确理解 |
|------|----------|
| ArkTS 等于 JavaScript | ArkTS 更强调静态类型和工程约束 |
| 所有变量变化都会刷新 UI | 只有被状态系统观察的变化才驱动 UI |
| `any` 很方便 | `any` 会绕开类型检查，降低可维护性 |
| 页面直接写所有业务 | 数据请求和业务规则应拆到类或 Repository |
| 忽略 key | 列表更新可能不稳定 |

## 24. 学习建议

学习顺序：

```text
let / const / 类型
  |
  v
函数 / 类 / 接口
  |
  v
泛型 / 联合类型 / 空值
  |
  v
Promise / async await
  |
  v
@Component / @State / build
  |
  v
列表、条件渲染、组件拆分
```

## 25. 参考资料

- ArkTS 入门：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-get-started
- ArkTS 语言基础：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-basic-syntax-overview
- ArkUI 状态管理：https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-state-management-overview
- ArkUI 组件：https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-components-summary

## 26. 总结

ArkTS 学习重点：

| 主题 | 价值 |
|------|------|
| 静态类型 | 减少运行时错误 |
| interface / type | 建模业务数据 |
| async / await | 处理异步任务 |
| ArkUI 装饰器 | 驱动页面状态刷新 |
| 组件拆分 | 提高复用和可维护性 |
| 列表和条件渲染 | 构建真实业务页面 |

不要只学语法本身。把 ArkTS 放到页面、状态、路由、网络请求、组件拆分里练习，学习效率最高。
