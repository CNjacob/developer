# JavaScript 与 TypeScript 核心知识

## 1. JavaScript 运行模型

JavaScript 单线程执行代码，宿主环境通过事件循环和异步 API 提供并发能力。

```text
同步代码 → Call Stack
异步操作 → Web / Native APIs
完成回调 → Microtask Queue / Task Queue
Event Loop → 栈为空时调度
```

一次事件循环通常执行一个宏任务，再清空微任务队列，浏览器随后视情况渲染。

| 类型 | 示例 |
| --- | --- |
| 同步任务 | 普通函数调用 |
| 微任务 | `Promise.then`、`queueMicrotask` |
| 宏任务 | `setTimeout`、I/O、UI 事件 |

```javascript
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
console.log("D");
// A D C B
```

`async/await` 是 Promise 的语法糖。`await` 后面的代码作为微任务继续执行，不会阻塞线程。

---

## 2. 作用域、闭包与执行上下文

JavaScript 使用词法作用域，变量可见性由声明位置决定。

| 声明 | 作用域 | 提升 | 重复声明 |
| --- | --- | --- | --- |
| `var` | 函数 | 提升并初始化为 `undefined` | 允许 |
| `let` | 块 | 提升但处于暂时性死区 | 不允许 |
| `const` | 块 | 提升但处于暂时性死区 | 不允许 |

`const` 只表示变量不能重新绑定，不表示对象内部不可修改。

闭包是函数与定义时词法环境的组合：

```javascript
function createCounter() {
  let count = 0;
  return () => ++count;
}
```

闭包常用于封装状态、回调、函数工厂和缓存。闭包长期持有大对象、DOM 或原生资源，也可能延长其生命周期。

---

## 3. `this` 与函数

普通函数的 `this` 由调用方式决定：

| 调用方式 | `this` |
| --- | --- |
| `obj.method()` | `obj` |
| `fn()` | 严格模式下为 `undefined` |
| `fn.call(obj)` | 显式指定的 `obj` |
| `new Fn()` | 新实例 |

箭头函数没有自己的 `this`、`arguments` 和 `prototype`，从外层词法环境捕获 `this`，不能作为构造函数。

`call`、`apply` 立即执行函数；`bind` 返回绑定后的新函数。

---

## 4. 原型与继承

对象读取属性时先查自身，再沿原型链向上查找：

```text
instance
  ↓ [[Prototype]]
Constructor.prototype
  ↓
Object.prototype
  ↓
null
```

`class` 主要是原型继承的语法封装。实例方法位于 `Constructor.prototype`，静态方法位于构造函数本身。

---

## 5. 类型、相等与拷贝

基本类型包括 `undefined`、`null`、`boolean`、`number`、`bigint`、`string`、`symbol`；数组、函数、Map、Set 等属于对象。

- `===`：不执行隐式类型转换，业务代码优先使用
- `==`：执行抽象相等比较，规则复杂
- `Object.is`：区分 `+0` 与 `-0`，认为 `NaN` 等于自身

展开语法和 `Object.assign` 只进行浅拷贝。`structuredClone` 可深拷贝支持结构化克隆的数据。JSON 序列化会丢失部分类型，也无法处理循环引用。

---

## 6. Promise 与异步控制

Promise 有 `pending`、`fulfilled`、`rejected` 三种状态，状态一旦确定不可逆。

| API | 行为 |
| --- | --- |
| `Promise.all` | 全部成功才成功，任意失败立即失败 |
| `Promise.allSettled` | 等待全部结束，保留每项结果 |
| `Promise.race` | 第一个确定状态的结果胜出 |
| `Promise.any` | 第一个成功结果胜出 |

工程实践：

- 无依赖任务并行执行，不要无意义地串行 `await`
- 用 `AbortController` 取消支持取消的请求
- 在系统边界统一转换错误，不要吞掉异常
- 限制批量并发，避免压垮服务端或设备

---

## 7. 模块化

ES Module 使用静态 `import/export`，便于依赖分析、Tree Shaking 和循环依赖检测。CommonJS 使用运行时 `require/module.exports`。

模块设计原则：

- 公开稳定、最小化的 API
- 避免跨层访问内部实现
- 保持依赖单向，避免循环依赖
- 业务模块按领域聚合，不只按文件类型分目录

---

## 8. TypeScript 的定位

TypeScript 在编译期提供静态类型检查，输出仍是 JavaScript。类型通常在运行时被擦除，因此网络响应、缓存和用户输入仍需运行时校验。

```text
TypeScript → 类型检查 → JavaScript → JS Engine
```

主要价值：

- 编译期发现错误
- 提供可靠重构能力
- 显式表达模块边界和业务模型
- 改善补全与代码导航

---

## 9. TypeScript 类型系统

### 9.1 `type` 与 `interface`

```typescript
interface User {
  id: string;
  name: string;
}

type UserState =
  | { status: "loading" }
  | { status: "loaded"; user: User }
  | { status: "failed"; error: Error };
```

- 对象公共契约、需要声明合并：常用 `interface`
- 联合、交叉、映射、条件类型：使用 `type`
- 团队一致性比机械区分更重要

### 9.2 `any`、`unknown`、`never`

- `any`：关闭类型检查，污染会沿调用链传播
- `unknown`：使用前必须缩窄，适合外部输入
- `never`：永不返回，或表示不可能出现的值

### 9.3 联合类型与类型缩窄

推荐用可辨识联合表达互斥状态，避免多个布尔值形成非法组合。

```typescript
type RequestState<T> =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: T }
  | { status: "error"; message: string };
```

可通过 `typeof`、`instanceof`、`in`、判别字段和自定义类型守卫缩窄类型。

### 9.4 泛型

泛型用于表达输入与输出的类型关系，而不是替代 `any`。

```typescript
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}

function getID<T extends { id: string }>(value: T): string {
  return value.id;
}
```

---

## 10. 高级类型

```typescript
const config = {
  environment: "production",
  retryCount: 3,
} as const;

type Config = typeof config;
type ConfigKey = keyof Config;
type Environment = Config["environment"];
```

常用工具类型：

| 类型 | 用途 |
| --- | --- |
| `Partial<T>` / `Required<T>` | 全部可选 / 必选 |
| `Readonly<T>` | 属性只读 |
| `Pick<T, K>` / `Omit<T, K>` | 选择 / 排除属性 |
| `Record<K, V>` | 键值映射 |
| `ReturnType<T>` | 函数返回类型 |
| `Awaited<T>` | 异步结果类型 |

条件类型与 `infer`：

```typescript
type ElementType<T> = T extends readonly (infer U)[] ? U : T;
```

- `as const`：收窄字面量并设为只读
- `satisfies`：校验目标类型，同时保留表达式的精确类型

---

## 11. 类型安全边界

类型断言只影响编译器，不执行运行时验证。不可信边界包括网络响应、本地存储、原生模块和用户输入。

```typescript
function isUser(value: unknown): value is User {
  if (typeof value !== "object" || value === null) return false;
  const object = value as Record<string, unknown>;
  return typeof object.id === "string" && typeof object.name === "string";
}
```

大型项目可使用 Schema 校验库，让运行时校验与类型推导保持一致。

---

## 12. 编译配置

新项目优先开启严格检查：

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "useUnknownInCatchVariables": true
  }
}
```

旧项目适合分阶段开启，并逐步消除错误。

---

## 13. 常见工程问题

- 防抖：停止触发一段时间后执行，适合搜索输入
- 节流：固定窗口最多执行一次，适合滚动和手势
- 不可变数据：便于比较、缓存和状态追踪，但要控制深层复制成本
- 内存泄漏：重点检查定时器、订阅、监听、闭包和无限缓存
- 循环依赖：抽取公共依赖、使用依赖倒置或重新划分边界

---

## 14. 执行上下文、提升与暂时性死区

每次执行全局代码或调用函数，JavaScript 都会创建执行上下文。执行上下文包含当前词法环境、变量环境和 `this` 绑定。

```text
调用函数
  ↓
创建执行上下文
  ├── 建立形参和局部变量绑定
  ├── 确定外部词法环境
  ├── 确定 this
  └── 执行函数体
  ↓
函数结束，执行上下文出栈
```

变量提升不是“代码移动”，而是声明在执行前已经被创建：

```javascript
console.log(value); // undefined
var value = 10;

// 可近似理解为：
var value;
console.log(value);
value = 10;
```

`let` 和 `const` 的绑定同样会提前创建，但在执行声明语句前不可访问，这段区域称为暂时性死区：

```javascript
{
  // console.log(name); // ReferenceError
  const name = "Ada";
  console.log(name);
}
```

函数声明整体可在声明位置之前调用；函数表达式只遵循承载变量的提升规则：

```javascript
sayHello(); // 正常

function sayHello() {
  console.log("hello");
}

// sayBye(); // TypeError: sayBye is not a function
var sayBye = function () {
  console.log("bye");
};
```

工程建议：

- 默认使用 `const`，确实需要重新赋值时使用 `let`
- 避免 `var`，减少函数作用域和重复声明带来的歧义
- 变量靠近使用位置声明，降低暂时性死区和阅读成本
- 不依赖函数或变量提升组织代码

---

## 15. 闭包的实际应用

### 15.1 私有状态

```javascript
function createWallet(initialBalance = 0) {
  let balance = initialBalance;

  return {
    deposit(amount) {
      if (amount <= 0) throw new Error("amount must be positive");
      balance += amount;
    },
    getBalance() {
      return balance;
    },
  };
}

const wallet = createWallet(100);
wallet.deposit(20);
console.log(wallet.getBalance()); // 120
```

外部无法直接修改 `balance`，只能通过返回的方法访问。

### 15.2 函数记忆化

```typescript
function memoize<Args extends readonly unknown[], Result>(
  fn: (...args: Args) => Result,
): (...args: Args) => Result {
  const cache = new Map<string, Result>();

  return (...args) => {
    const key = JSON.stringify(args);
    const cached = cache.get(key);
    if (cached !== undefined) return cached;

    const result = fn(...args);
    cache.set(key, result);
    return result;
  };
}
```

真实项目中要考虑键的稳定性、缓存上限、过期时间和对象参数，不能让缓存无限增长。

### 15.3 循环中的闭包

```javascript
for (var i = 0; i < 3; i += 1) {
  setTimeout(() => console.log(i), 0);
}
// 3 3 3

for (let i = 0; i < 3; i += 1) {
  setTimeout(() => console.log(i), 0);
}
// 0 1 2
```

`var` 只创建一个函数级绑定；`let` 会为每轮循环创建新的词法绑定。

---

## 16. 原型、构造函数与类

### 16.1 `prototype` 与 `[[Prototype]]`

- 函数的 `prototype` 属性：实例创建时使用的原型对象
- 对象的 `[[Prototype]]`：对象实际委托属性查找的内部链接
- `Object.getPrototypeOf(obj)`：读取对象原型的标准方法

```javascript
function Person(name) {
  this.name = name;
}

Person.prototype.greet = function () {
  return `Hello, ${this.name}`;
};

const person = new Person("Ada");

console.log(Object.getPrototypeOf(person) === Person.prototype); // true
console.log(person.greet()); // Hello, Ada
```

`new Person("Ada")` 大致经历：

1. 创建空对象
2. 把对象原型设为 `Person.prototype`
3. 以新对象作为 `this` 调用 `Person`
4. 构造函数未显式返回对象时，返回新对象

### 16.2 `class` 继承

```typescript
abstract class Shape {
  constructor(readonly color: string) {}
  abstract area(): number;
}

class Circle extends Shape {
  constructor(
    color: string,
    private readonly radius: number,
  ) {
    super(color);
  }

  area(): number {
    return Math.PI * this.radius ** 2;
  }
}
```

组合通常比深继承层次更易维护。继承适合稳定的“是一个”关系，功能拼装更适合组合和接口。

---

## 17. 异步编程完整示例

### 17.1 超时、取消与错误分类

```typescript
class HttpError extends Error {
  constructor(
    readonly status: number,
    message: string,
  ) {
    super(message);
    this.name = "HttpError";
  }
}

async function requestJSON<T>(
  url: string,
  options: RequestInit & { timeout?: number } = {},
): Promise<T> {
  const controller = new AbortController();
  const timer = setTimeout(
    () => controller.abort(new Error("request timeout")),
    options.timeout ?? 10_000,
  );

  try {
    const response = await fetch(url, {
      ...options,
      signal: controller.signal,
    });

    if (!response.ok) {
      throw new HttpError(response.status, `HTTP ${response.status}`);
    }

    return (await response.json()) as T;
  } finally {
    clearTimeout(timer);
  }
}
```

此处的 `as T` 只提供编译期类型，生产代码还应对 JSON 做运行时校验。

### 17.2 并行与串行

```typescript
// 两个请求互不依赖，应并行
const [profile, permissions] = await Promise.all([
  loadProfile(),
  loadPermissions(),
]);

// 第二步依赖第一步结果，必须串行
const order = await createOrder();
const payment = await payOrder(order.id);
```

### 17.3 限制并发

```typescript
async function mapWithConcurrency<T, R>(
  values: readonly T[],
  concurrency: number,
  transform: (value: T) => Promise<R>,
): Promise<R[]> {
  const results = new Array<R>(values.length);
  let nextIndex = 0;

  async function worker(): Promise<void> {
    while (nextIndex < values.length) {
      const index = nextIndex++;
      results[index] = await transform(values[index]);
    }
  }

  const workerCount = Math.min(concurrency, values.length);
  await Promise.all(Array.from({ length: workerCount }, worker));
  return results;
}
```

它适用于批量上传、图片处理等场景。实际实现还要定义单项失败时是立即终止还是收集全部结果。

---

## 18. 数组、对象与函数式数据处理

常用数组方法：

| 方法 | 返回值 | 是否修改原数组 |
| --- | --- | --- |
| `map` | 转换后的新数组 | 否 |
| `filter` | 满足条件的新数组 | 否 |
| `reduce` | 累积结果 | 否 |
| `find` | 第一项匹配值 | 否 |
| `some` / `every` | 布尔值 | 否 |
| `sort` | 排序后的原数组 | 是 |
| `toSorted` | 排序后的新数组 | 否 |

```typescript
interface Order {
  id: string;
  status: "pending" | "paid" | "cancelled";
  amount: number;
}

const paidTotal = orders
  .filter((order) => order.status === "paid")
  .reduce((sum, order) => sum + order.amount, 0);
```

避免为了“函数式”创建过多临时数组。超大数据集合可以用单次循环或生成器减少分配。

对象不可变更新：

```typescript
const nextUser = {
  ...user,
  profile: {
    ...user.profile,
    nickname: "Grace",
  },
};
```

展开操作只复制路径上的对象。嵌套层级过深通常意味着状态模型需要扁平化。

---

## 19. TypeScript 建模实战

### 19.1 避免布尔值组合

不推荐：

```typescript
interface BadState {
  isLoading: boolean;
  isSuccess: boolean;
  data?: User;
  error?: string;
}
```

它允许 `isLoading` 和 `isSuccess` 同时为 `true`。推荐：

```typescript
type LoadState<T> =
  | { type: "idle" }
  | { type: "loading" }
  | { type: "success"; data: T }
  | { type: "failure"; error: AppError };
```

### 19.2 穷尽检查

```typescript
function stateText<T>(state: LoadState<T>): string {
  switch (state.type) {
    case "idle":
      return "尚未加载";
    case "loading":
      return "加载中";
    case "success":
      return "加载成功";
    case "failure":
      return state.error.message;
    default: {
      const impossible: never = state;
      return impossible;
    }
  }
}
```

新增联合成员但遗漏分支时，编译器会在 `never` 赋值处报错。

### 19.3 品牌类型

结构类型系统中，两个 `string` 默认可互换。品牌类型可区分业务 ID：

```typescript
type Brand<Value, Name extends string> = Value & {
  readonly __brand: Name;
};

type UserID = Brand<string, "UserID">;
type OrderID = Brand<string, "OrderID">;

function loadUser(id: UserID): Promise<User> {
  return userRepository.find(id);
}
```

品牌类型只提供编译期约束，创建品牌值时仍需要在系统边界验证格式。

### 19.4 泛型 API 结果

```typescript
type Result<Value, Failure> =
  | { ok: true; value: Value }
  | { ok: false; error: Failure };

async function safeRequest<T>(
  operation: () => Promise<T>,
): Promise<Result<T, AppError>> {
  try {
    return { ok: true, value: await operation() };
  } catch (error: unknown) {
    return { ok: false, error: normalizeError(error) };
  }
}
```

`Result` 适合期望调用者显式处理的业务失败。真正不可恢复的编程错误不应全部转换为普通结果。

---

## 20. 类型兼容、方差与函数类型

TypeScript 主要采用结构类型：只要结构满足要求，就可以赋值。

```typescript
interface Named {
  name: string;
}

const detailed = { name: "Ada", age: 36 };
const named: Named = detailed; // 合法
```

直接传入对象字面量会执行额外属性检查，帮助发现拼写错误。

函数类型需要特别注意参数方向：

```typescript
type AnimalHandler = (animal: Animal) => void;
type DogHandler = (dog: Dog) => void;

// strictFunctionTypes 下，不能把只会处理 Dog 的函数
// 当成能够处理任意 Animal 的函数。
```

- 协变：输出类型可以更具体
- 逆变：输入类型需要能处理更宽泛的值
- 不变：两个方向都不能安全替换

理解方差有助于设计事件处理器、泛型容器和回调 API。

---

## 21. 声明文件与模块边界

没有类型定义的 JavaScript 库可通过 `.d.ts` 描述：

```typescript
declare module "legacy-sdk" {
  export interface LoginResult {
    token: string;
  }

  export function login(
    username: string,
    password: string,
  ): Promise<LoginResult>;
}
```

不要在业务代码到处使用 `declare` 掩盖真实不确定性。声明文件必须与运行时实现一致，否则只会制造虚假的类型安全。

模块公开入口：

```typescript
// features/account/index.ts
export type { Account, AccountID } from "./model";
export { createAccountService } from "./service";
```

外部只从 `index.ts` 访问模块，内部目录可自由重构。

---

## 22. 测试与可测试设计

纯函数最容易测试：

```typescript
export function calculateDiscount(
  amount: number,
  level: "normal" | "vip",
): number {
  if (amount < 0) throw new Error("invalid amount");
  return level === "vip" ? amount * 0.8 : amount;
}
```

依赖外部系统的逻辑使用依赖注入：

```typescript
interface Clock {
  now(): Date;
}

class CouponService {
  constructor(private readonly clock: Clock) {}

  isExpired(expireAt: Date): boolean {
    return expireAt.getTime() <= this.clock.now().getTime();
  }
}
```

测试可传入固定时钟，无需修改系统时间。随机数、网络、存储和设备信息也应通过边界注入。

测试层次：

- 单元测试：纯函数、业务规则和状态转换
- 集成测试：API Client、存储、模块协作
- 端到端测试：关键用户链路

---

## 23. 完整模块示例

```text
features/user/
├── model.ts
├── schema.ts
├── repository.ts
├── service.ts
└── index.ts
```

```typescript
// model.ts
export interface User {
  id: string;
  name: string;
}

export type UserError =
  | { type: "notFound" }
  | { type: "network"; message: string }
  | { type: "invalidData" };

// repository.ts
export interface UserRepository {
  find(id: string): Promise<User>;
}

// service.ts
export class UserService {
  constructor(private readonly repository: UserRepository) {}

  async displayName(id: string): Promise<string> {
    const user = await this.repository.find(id);
    return user.name.trim() || "未命名用户";
  }
}
```

Model 表达领域数据，Repository 隔离数据来源，Service 编排业务规则。UI 不需要知道用户来自网络还是缓存。

---

## 24. 总结

| 主题 | 核心结论 |
| --- | --- |
| 运行模型 | 单线程执行，事件循环调度异步任务 |
| 闭包 | 保存词法环境，也可能延长对象生命周期 |
| 原型 | JavaScript 继承和属性查找的基础 |
| Promise | 表达异步结果，微任务驱动后续回调 |
| 模块 | 依赖边界应稳定、单向、最小化 |
| TypeScript | 提供编译期保障，不替代运行时校验 |
| 类型建模 | 优先用联合类型表达合法状态 |
| 工程配置 | 严格检查是长期类型安全的基础 |
