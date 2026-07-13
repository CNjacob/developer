# React Native 核心知识

## 1. React Native 的定位

React Native 使用 React 的声明式模型，通过 JavaScript/TypeScript 描述状态和界面，并渲染为 iOS、Android 原生视图。

```text
React + TypeScript
        ↓
React Native Runtime
        ↓
原生组件与平台 API
        ↓
iOS / Android
```

它不是 WebView 方案。适合双端业务高度相似、需要快速迭代的标准业务页面；重度音视频、3D、专业图形编辑和大量平台独有交互需要谨慎评估。

---

## 2. React 核心模型

React UI 可以视为状态的函数：

```text
UI = f(props, state)
```

- Props：父组件传入的只读数据
- State：组件拥有、能够触发渲染的数据
- Context：跨层共享、相对低频变化的数据

状态原则：

- 放在真正拥有它的最近公共父节点
- 可从现有数据计算的值不重复存储
- 服务端状态与本地 UI 状态分开
- 不用 Context 承载高频变化的大对象

常用 Hooks：

| Hook | 作用 |
| --- | --- |
| `useState` | 局部状态 |
| `useReducer` | 复杂状态转换 |
| `useEffect` | 与外部系统同步 |
| `useMemo` | 缓存计算结果 |
| `useCallback` | 缓存函数引用 |
| `useRef` | 保存跨渲染值或访问实例 |

`useEffect` 适合网络、订阅、定时器和原生资源。依赖数组应包含 Effect 使用的响应式值，清理函数负责取消订阅和释放资源。

---

## 3. 渲染与更新

```text
用户事件 / 数据返回
        ↓
状态更新
        ↓
Render：计算新元素树
        ↓
Reconciliation：比较差异
        ↓
Commit：提交原生变更
        ↓
平台绘制
```

组件重新渲染不等于原生视图重建。列表 `key` 用于标识元素身份，应使用稳定业务 ID。

`React.memo`、`useMemo`、`useCallback` 是性能工具，不应默认添加。只有渲染成本明显，或稳定引用确实能减少子组件更新时才有收益。

---

## 4. 线程模型

| 线程 | 职责 |
| --- | --- |
| JS Thread | 执行 JS、React 和业务逻辑 |
| UI/Main Thread | 原生视图、触摸和平台绘制 |
| Native Module Queue | 执行指定原生模块工作 |
| Render Thread | 部分平台的底层绘制工作 |

JS 线程长任务会导致事件、状态计算和 JS 驱动动画卡顿；主线程繁忙会影响绘制和触摸。优化前必须先判断瓶颈线程。

---

## 5. 旧架构与新架构

旧架构通过异步 Bridge 传递序列化消息：

```text
JavaScript → 序列化 → Bridge → Native
```

其局限包括序列化成本、细粒度跨端调用效率低、模块通常异步访问、类型契约依赖人工维护。

新架构的核心组成：

| 组件 | 职责 |
| --- | --- |
| JSI | JS 与 C++/原生对象交互的接口 |
| Fabric | 新渲染系统 |
| TurboModules | 支持懒加载和类型生成的原生模块系统 |
| Codegen | 根据类型规范生成跨语言胶水代码 |
| Hermes | 面向 RN 优化的 JavaScript 引擎 |

```text
React → Fabric → C++ Shadow Tree → Native View
JavaScript ↔ JSI ↔ TurboModules / Platform APIs
```

新架构减少传统 Bridge 的序列化和批量通信限制，但跨语言调用仍有成本。高频或大数据交互仍应批处理或下沉计算。

---

## 6. Fabric 渲染

Fabric 使用 C++ Shadow Tree 表达 UI：

1. React 计算组件变化
2. Renderer 创建不可变 Shadow Tree
3. Layout 阶段计算布局
4. Commit 阶段生成新版本
5. Mount 阶段把差异应用到原生视图

不可变结构便于安全构建和比较 UI。并发渲染可以中断或放弃未提交工作，只有提交结果影响原生界面。

---

## 7. 布局、样式与资源

RN 通常使用 Yoga 计算 Flexbox 布局，与 Web CSS 不完全相同：

- 默认主轴通常为纵向
- 属性使用驼峰命名
- 没有完整 CSS 层叠和选择器
- 数值尺寸通常是逻辑像素
- 文本必须放在 `Text` 中

工程中应统一颜色、间距、字号、圆角和阴影 Token。

图片优化：

- 声明合理尺寸，避免解码超大图
- 列表使用缩略图
- 使用缓存和预加载
- 根据平台选择格式
- 避免频繁切换未缓存远程图片

---

## 8. 导航与生命周期

需要区分：

- React 组件挂载/卸载
- 页面获得/失去焦点
- App 前台/后台
- ViewController/Activity 生命周期

页面失焦不一定卸载。摄像头、播放器、定位和定时器等资源应根据真实可见性和 App 状态暂停或释放。

---

## 9. 状态管理

| 状态类型 | 示例 | 推荐归属 |
| --- | --- | --- |
| 组件状态 | 输入框、弹窗 | `useState` / `useReducer` |
| 页面状态 | 筛选、编辑草稿 | 页面 Hook 或 Store |
| 服务端状态 | 列表、缓存、重试 | Query/缓存层 |
| 全局状态 | 登录态、主题 | 小型 Store / Context |
| 导航状态 | 路由、参数 | 导航系统 |

常见问题：

- 所有数据都放进全局 Store
- 将服务端缓存复制到多个 Store
- 多个布尔值表达互斥状态
- Selector 每次返回新对象
- 组件直接编排复杂业务和请求

推荐用可辨识联合表达请求状态，让状态转换保持单向、可追踪。

---

## 10. 网络与数据层

```text
Screen / Component
      ↓
UseCase / Feature Service
      ↓
Repository
  ↙         ↘
Remote API   Local Storage
```

- API Client：请求、认证、序列化、错误标准化
- Repository：隐藏数据来源，协调缓存与远端
- UseCase：表达业务操作和规则
- UI：发出意图并渲染状态

外部响应必须运行时校验，不能只做 TypeScript 类型断言。请求层还应统一处理超时、取消、重试、鉴权刷新和监控。

---

## 11. 原生模块

适合下沉原生的能力：

- 系统 API 和原生 SDK
- 音视频、蓝牙、传感器
- 密集计算
- 原生 UI 组件
- 对启动或交互延迟敏感的基础能力

设计原则：

- API 以业务能力为单位
- 输入输出可验证、可版本化
- 明确线程、同步/异步和回调语义
- 明确资源所有权与释放时机
- 双端错误码和行为一致
- 使用 Codegen 减少契约漂移

Native Component 用于封装原生 UI；TurboModule 用于暴露非 UI 平台能力。

---

## 12. 性能优化

先测量，再定位：

```text
明确现象
  ↓
测量 JS/UI FPS、CPU、内存、网络
  ↓
确认线程和触发场景
  ↓
最小范围优化
  ↓
对比前后数据
```

列表：

- 使用虚拟化列表和稳定 `key`
- 减少 Cell 层级与昂贵计算
- 调整窗口和批量渲染参数
- 图片缩略、缓存和预取
- 分页加载

渲染：

- 将状态放在最小更新范围
- 拆分高频变化和静态区域
- 避免在 render 中做重计算
- 只对确认昂贵的组件记忆化
- 动画优先在 UI 线程执行

启动与 JS：

- 拆分长任务
- 批处理跨端通信
- 避免同步读取大量存储
- 延迟非关键初始化
- 分阶段测量原生启动、JS 加载和首屏

内存重点检查监听、定时器、订阅、导航栈、大图、缓存、闭包和未释放的原生资源。

---

## 13. 工程架构

推荐按业务 Feature 聚合：

```text
src/
├── app/                 # 启动、导航、依赖装配
├── features/
│   ├── account/
│   │   ├── api/
│   │   ├── components/
│   │   ├── screens/
│   │   └── model/
│   └── order/
├── shared/
│   ├── ui/
│   ├── network/
│   ├── storage/
│   └── utils/
└── native/              # 原生能力的 TS 适配层
```

依赖保持 `app → features → shared`。Feature 不直接引用其他 Feature 的内部文件，跨 Feature 流程由应用层编排。

质量保障：

- TypeScript 严格模式
- ESLint 与统一格式化
- 单元测试覆盖纯业务逻辑
- 组件测试覆盖关键交互
- E2E 覆盖核心链路
- CI 执行类型检查、Lint、测试和双端构建

---

## 14. 混合架构

```text
Native App
  ├── Navigation / Lifecycle
  ├── Account / Payment / Platform Services
  └── RN Container
        ├── RN Features
        └── Typed Native Adapter
```

混合项目需统一：

- 路由和返回协议
- 登录态与环境配置
- 埋点、日志和错误监控
- 网络与鉴权策略
- 主题、语言和无障碍
- Bundle 发布、灰度和回滚

JS 层不应绕过适配层直接依赖大量平台实现，否则双端行为会逐渐分叉。

---

## 15. 选型结论

RN 更准确的目标是最大化共享业务代码，同时保留平台适配能力，而不是“一次开发完全无需双端工作”。

选型应综合评估：

- 业务共享率
- 交互复杂度和性能上限
- 团队的 React 与原生能力
- 已有原生资产
- 发布模式与稳定性要求
- 首版速度和长期维护成本

---

## 16. 组件设计示例

### 16.1 展示组件与容器组件

展示组件只根据输入渲染：

```tsx
interface UserCardProps {
  name: string;
  avatarURL?: string;
  onPress: () => void;
}

export const UserCard = React.memo(function UserCard({
  name,
  avatarURL,
  onPress,
}: UserCardProps) {
  return (
    <Pressable
      accessibilityRole="button"
      accessibilityLabel={`查看 ${name} 的资料`}
      onPress={onPress}
      style={styles.card}
    >
      <Image source={{ uri: avatarURL }} style={styles.avatar} />
      <Text numberOfLines={1} style={styles.name}>
        {name}
      </Text>
    </Pressable>
  );
});
```

容器负责数据和行为：

```tsx
function UserCardContainer({ userID }: { userID: string }) {
  const navigation = useNavigation();
  const user = useUser(userID);

  const openProfile = useCallback(() => {
    navigation.navigate("Profile", { userID });
  }, [navigation, userID]);

  if (user.status === "loading") return <UserCardSkeleton />;
  if (user.status === "error") return <ErrorView message={user.message} />;

  return (
    <UserCard
      name={user.data.name}
      avatarURL={user.data.avatarURL}
      onPress={openProfile}
    />
  );
}
```

不是每个组件都必须机械拆分。只有展示可复用、数据逻辑复杂或需要单独测试时，这种拆分才有价值。

### 16.2 受控输入

```tsx
interface SearchBarProps {
  value: string;
  onChange: (value: string) => void;
  onSubmit: () => void;
}

function SearchBar({ value, onChange, onSubmit }: SearchBarProps) {
  return (
    <TextInput
      value={value}
      onChangeText={onChange}
      onSubmitEditing={onSubmit}
      returnKeyType="search"
      placeholder="搜索"
    />
  );
}
```

受控组件的状态由外部持有，便于校验、联动和重置；高频大型表单也要关注每次输入造成的渲染范围。

---

## 17. Hooks 深入

### 17.1 自定义 Hook

```typescript
type LoadState<T> =
  | { type: "loading" }
  | { type: "success"; data: T }
  | { type: "failure"; message: string };

function useUser(userID: string): LoadState<User> {
  const [state, setState] = useState<LoadState<User>>({
    type: "loading",
  });

  useEffect(() => {
    const controller = new AbortController();
    setState({ type: "loading" });

    userRepository
      .find(userID, controller.signal)
      .then((data) => setState({ type: "success", data }))
      .catch((error: unknown) => {
        if (!controller.signal.aborted) {
          setState({ type: "failure", message: toMessage(error) });
        }
      });

    return () => controller.abort();
  }, [userID]);

  return state;
}
```

关键点：

- `userID` 变化时取消旧请求
- 卸载后不再提交过期结果
- Hook 返回领域状态，不泄漏底层请求细节
- Effect 只负责同步外部系统

### 17.2 陈旧闭包

```tsx
function Counter() {
  const [count, setCount] = useState(0);

  const incrementLater = () => {
    setTimeout(() => {
      // 使用函数式更新，避免读取创建回调时的旧 count
      setCount((current) => current + 1);
    }, 1000);
  };

  return <Button title={`${count}`} onPress={incrementLater} />;
}
```

每次渲染都有独立的 props、state 和函数闭包。异步回调读取的是创建它那次渲染的数据。

### 17.3 `useReducer`

```typescript
interface FormState {
  email: string;
  password: string;
  submitting: boolean;
  error?: string;
}

type FormAction =
  | { type: "emailChanged"; value: string }
  | { type: "passwordChanged"; value: string }
  | { type: "submitStarted" }
  | { type: "submitFailed"; message: string };

function reducer(state: FormState, action: FormAction): FormState {
  switch (action.type) {
    case "emailChanged":
      return { ...state, email: action.value, error: undefined };
    case "passwordChanged":
      return { ...state, password: action.value, error: undefined };
    case "submitStarted":
      return { ...state, submitting: true, error: undefined };
    case "submitFailed":
      return { ...state, submitting: false, error: action.message };
  }
}
```

当多个字段围绕同一状态机变化时，Reducer 比多个独立 `useState` 更容易追踪。

---

## 18. 列表实现与优化示例

```tsx
interface Message {
  id: string;
  title: string;
  summary: string;
}

const MessageRow = React.memo(function MessageRow({
  item,
  onPress,
}: {
  item: Message;
  onPress: (id: string) => void;
}) {
  return (
    <Pressable onPress={() => onPress(item.id)} style={styles.row}>
      <Text style={styles.title}>{item.title}</Text>
      <Text numberOfLines={2}>{item.summary}</Text>
    </Pressable>
  );
});

function MessageList({ messages }: { messages: readonly Message[] }) {
  const openMessage = useCallback((id: string) => {
    router.openMessage(id);
  }, []);

  const renderItem = useCallback(
    ({ item }: { item: Message }) => (
      <MessageRow item={item} onPress={openMessage} />
    ),
    [openMessage],
  );

  return (
    <FlatList
      data={messages}
      keyExtractor={(item) => item.id}
      renderItem={renderItem}
      initialNumToRender={12}
      windowSize={7}
      removeClippedSubviews
    />
  );
}
```

优化判断：

- `keyExtractor` 必须稳定
- Row 的 Props 应尽量小且稳定
- 固定高度列表可提供 `getItemLayout`
- `windowSize` 过小可能出现空白，过大增加内存
- `removeClippedSubviews` 需要在复杂变换和无障碍场景验证
- 不要仅为稳定引用加入大量 `useCallback`，应通过性能数据判断

分页状态应独立表达首屏加载、加载更多、刷新和错误，避免一个 `isLoading` 控制全部场景。

---

## 19. 导航与参数类型

```typescript
type RootStackParamList = {
  Home: undefined;
  Profile: { userID: string };
  EditProfile: { userID: string; source?: "profile" | "settings" };
};
```

页面只接收完成渲染所需的最小参数，通常传稳定 ID，而不是把大型可变对象塞进路由。

导航职责建议：

- Screen 发出“查看资料”“完成支付”等业务意图
- Feature Router 转换为具体路由操作
- App Router 处理跨 Feature 流程、登录拦截和 Deep Link

Deep Link 需要统一完成 URL 解析、参数校验、登录检查、页面构建和失败降级。

---

## 20. 网络与缓存完整示例

```typescript
interface UserDTO {
  id: unknown;
  display_name: unknown;
}

function decodeUser(value: unknown): User {
  if (typeof value !== "object" || value === null) {
    throw new Error("invalid user");
  }

  const dto = value as UserDTO;
  if (typeof dto.id !== "string" || typeof dto.display_name !== "string") {
    throw new Error("invalid user fields");
  }

  return { id: dto.id, name: dto.display_name };
}

class RemoteUserRepository implements UserRepository {
  constructor(private readonly client: HttpClient) {}

  async find(id: string, signal?: AbortSignal): Promise<User> {
    const json = await this.client.get(`/users/${id}`, { signal });
    return decodeUser(json);
  }
}
```

带缓存的 Repository：

```typescript
class CachedUserRepository implements UserRepository {
  constructor(
    private readonly remote: UserRepository,
    private readonly cache: UserCache,
  ) {}

  async find(id: string, signal?: AbortSignal): Promise<User> {
    const cached = await this.cache.get(id);
    if (cached) return cached;

    const user = await this.remote.find(id, signal);
    await this.cache.set(user);
    return user;
  }
}
```

缓存策略必须说明：

- 读取顺序：cache-first、network-first 或 stale-while-revalidate
- 过期时间
- 写入失败是否影响主流程
- 登录退出时清理哪些数据
- 多请求同时 miss 时是否合并请求

---

## 21. 原生模块契约示例

以设备安全能力为例，TypeScript 层先定义稳定业务接口：

```typescript
export interface SecureDeviceService {
  deviceID(): Promise<string>;
  sign(payload: string): Promise<string>;
}
```

TurboModule Spec 示例：

```typescript
import type { TurboModule } from "react-native";
import { TurboModuleRegistry } from "react-native";

export interface Spec extends TurboModule {
  getDeviceID(): Promise<string>;
  sign(payload: string): Promise<string>;
}

export default TurboModuleRegistry.getEnforcing<Spec>(
  "SecureDeviceModule",
);
```

业务层再通过 Adapter 隔离生成代码：

```typescript
class NativeSecureDeviceService implements SecureDeviceService {
  async deviceID(): Promise<string> {
    const value = await SecureDeviceModule.getDeviceID();
    if (!value) throw new Error("empty device id");
    return value;
  }

  sign(payload: string): Promise<string> {
    return SecureDeviceModule.sign(payload);
  }
}
```

设计时应明确：

- 方法在哪条线程执行
- Promise 在什么条件下成功或失败
- 错误 code、message 和可恢复性
- 是否允许并发调用
- 模块不可用时如何降级
- 传输大数据时是否改用文件路径或批量接口

---

## 22. 原生 UI 组件边界

原生 UI 组件适合播放器、地图、相机预览等平台视图：

```tsx
interface NativePlayerProps {
  sourceURL: string;
  paused: boolean;
  onStateChange?: (event: {
    nativeEvent: { state: "idle" | "playing" | "failed" };
  }) => void;
}
```

Props 表达期望状态，Event 表达原生发生的事实。不要让 JS 同时发送命令和 Props 控制同一状态，否则容易产生双数据源。

资源生命周期：

```text
组件创建 → 创建原生资源
Props 更新 → 更新播放器状态
页面失焦 → 暂停或降级
组件卸载 → 取消任务并释放资源
App 后台 → 按产品规则暂停
```

---

## 23. 动画与手势

动画是否流畅取决于每帧工作运行在哪条线程。需要持续读取 JS 状态的动画容易受 JS 长任务影响。

推荐原则：

- 简单进入/退出动画使用平台或 RN 声明式能力
- 连续手势和高频动画尽量在 UI 线程计算
- 避免动画过程中反复触发 React 大范围渲染
- 位移和透明度通常比引发布局的属性成本更低
- 动画完成后再同步最终业务状态

```text
Gesture Event
    ↓ UI Thread
Shared Animation State
    ↓
Transform / Opacity
    ↓ 动画结束
JS Business State
```

手势冲突需要明确父子滚动、点击、横滑返回和系统手势的优先级。

---

## 24. 启动性能拆解

不要只记录一个“启动耗时”，应分阶段测量：

```text
进程启动
  ↓ Native 初始化
RN Runtime 创建
  ↓ JS Bundle 加载
JS 执行与模块初始化
  ↓ React 首次 Render
Fabric Commit / Mount
  ↓
首屏可见与可交互
```

常见优化：

- 首屏前只初始化必要 SDK
- 避免模块导入时执行重任务
- 延迟日志上传、预加载和非关键请求
- 控制 Bundle 体积与第三方依赖
- 首屏数据优先使用缓存或骨架屏
- 对预编译字节码、Bundle 分包和 OTA 策略进行真实设备验证

指标至少区分冷启动、热启动、首屏可见和首屏可交互。

---

## 25. 错误处理与可观测性

错误按层转换：

```text
Native / HTTP Error
        ↓
Infrastructure Error
        ↓
Domain Error
        ↓
User-facing State
```

例如 HTTP 401 不应直接在每个页面显示“401”，而应由鉴权层刷新 Token 或触发登录流程。

日志应包含：

- trace/request ID
- RN Bundle 和 App 版本
- 平台、系统和设备信息
- 页面与用户操作阶段
- 原生错误 code 和 JS stack
- 关键耗时与结果

不得记录 Token、密码、完整身份证号等敏感信息。错误边界用于提供 UI 降级，但不能代替底层异常监控。

---

## 26. 测试策略与示例

组件测试关注用户可见行为：

```tsx
it("submits credentials", async () => {
  const login = jest.fn().mockResolvedValue(undefined);
  const screen = render(<LoginForm login={login} />);

  fireEvent.changeText(screen.getByLabelText("邮箱"), "a@example.com");
  fireEvent.changeText(screen.getByLabelText("密码"), "secret");
  fireEvent.press(screen.getByRole("button", { name: "登录" }));

  await waitFor(() => {
    expect(login).toHaveBeenCalledWith("a@example.com", "secret");
  });
});
```

测试金字塔：

| 层次 | 重点 |
| --- | --- |
| 单元测试 | Reducer、UseCase、Mapper、Validator |
| 组件测试 | 渲染状态和用户交互 |
| 原生集成测试 | TurboModule、Native Component 契约 |
| E2E | 登录、支付、核心交易链路 |

不要过度依赖巨大 Snapshot。它们容易产生低价值更新，关键状态和交互应使用明确断言。

---

## 27. Feature 模块完整示例

```text
features/profile/
├── api/
│   ├── ProfileDTO.ts
│   └── RemoteProfileRepository.ts
├── domain/
│   ├── Profile.ts
│   ├── ProfileRepository.ts
│   └── LoadProfile.ts
├── presentation/
│   ├── ProfileScreen.tsx
│   ├── ProfileViewModel.ts
│   └── components/
├── navigation/
│   └── ProfileRouter.ts
└── index.ts
```

数据流：

```text
ProfileScreen
    ↓ load(userID)
ProfileViewModel / Hook
    ↓ execute(userID)
LoadProfile UseCase
    ↓ find(userID)
ProfileRepository
    ↓
HTTP / Cache
```

依赖装配：

```typescript
const remote = new RemoteProfileRepository(httpClient);
const repository = new CachedProfileRepository(remote, profileCache);
const loadProfile = new LoadProfile(repository);

export function ProfileRoute({ userID }: { userID: string }) {
  return <ProfileScreen userID={userID} loadProfile={loadProfile} />;
}
```

组装发生在应用边界，领域和展示层依赖抽象。测试时可注入内存 Repository。

---

## 28. RN 项目评审清单

| 模块 | 检查项 |
| --- | --- |
| 组件 | Props 是否最小、状态归属是否正确 |
| Hooks | Effect 是否有完整依赖和清理 |
| 列表 | Key、图片、分页和渲染窗口是否合理 |
| 状态 | 服务端状态是否与 UI 状态分离 |
| 网络 | 是否支持取消、超时、校验和错误转换 |
| 导航 | 参数是否类型安全，生命周期是否正确 |
| Native | 线程、错误、资源所有权是否明确 |
| 性能 | 是否有真实测量，瓶颈线程是否确认 |
| 稳定性 | 是否有监控、降级、灰度和回滚 |
| 测试 | 业务规则、关键交互、跨端契约是否覆盖 |

---

## 29. 总结

| 主题 | 核心结论 |
| --- | --- |
| 编程模型 | React 以状态驱动声明式 UI |
| 渲染 | Render 计算变化，Commit 应用原生视图 |
| 新架构 | JSI、Fabric、TurboModules、Codegen 是核心 |
| 状态 | 按性质分层，避免无边界全局 Store |
| 原生通信 | API 业务化、类型化、批处理 |
| 性能 | 先识别 JS 还是 UI 线程瓶颈 |
| 工程架构 | 按 Feature 聚合，依赖保持单向 |
| 技术选型 | 追求合理共享，保留平台差异 |
