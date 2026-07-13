# iOS RunLoop 底层原理

## 1. RunLoop 是什么

RunLoop 是线程的事件循环机制。它让线程在有任务时处理任务，在没有任务时休眠等待，而不是执行完就退出或空转消耗 CPU。

简化模型：

```c
while (running) {
    event = waitForEvent();
    handle(event);
}
```

RunLoop 的价值：

- 保持线程存活
- 处理事件源
- 管理定时器
- 在线程空闲时休眠
- 被事件唤醒后继续处理
- 配合 AutoreleasePool、触摸事件、渲染提交

iOS 中有两套 API：

| API | 层级 |
| --- | --- |
| `NSRunLoop` | Foundation 面向对象封装 |
| `CFRunLoopRef` | CoreFoundation C 接口 |

`NSRunLoop` 是对 `CFRunLoopRef` 的封装。

---

## 2. RunLoop 与线程

### 2.1 一一对应

RunLoop 和线程是一一对应的：

```text
Main Thread
  └── Main RunLoop，系统自动创建并运行

Sub Thread
  └── RunLoop 懒加载创建，但默认不运行
```

规则：

- 主线程 RunLoop 由系统启动
- 子线程 RunLoop 首次获取时创建
- 子线程 RunLoop 不会自动运行
- 线程结束后，对应 RunLoop 也失去意义

### 2.2 子线程 RunLoop

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
    [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];
    [runLoop run];
});
```

如果子线程没有输入源或 Timer，RunLoop 会直接退出。因此通常需要添加 Port、Source 或 Timer 维持运行。

---

## 3. RunLoop 对象模型

RunLoop 包含多个 Mode，每个 Mode 中包含 Source、Timer、Observer。

```text
CFRunLoop
  ├── currentMode
  ├── modes
  │   ├── kCFRunLoopDefaultMode
  │   │   ├── sources
  │   │   ├── timers
  │   │   └── observers
  │   └── UITrackingRunLoopMode
  └── commonModes
```

### 3.1 Mode

Mode 是 RunLoop 的运行场景。RunLoop 每次只能运行在一个 Mode 中。

常见 Mode：

| Mode | 场景 |
| --- | --- |
| `kCFRunLoopDefaultMode` | 默认模式，普通事件处理 |
| `UITrackingRunLoopMode` | ScrollView 滚动追踪 |
| `kCFRunLoopCommonModes` | 不是真实 Mode，是一组 Mode 标记 |

滚动时 Timer 不触发，通常是因为 Timer 只添加到了 Default Mode。

```objc
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

### 3.2 Source

Source 是事件源。

| 类型 | 说明 | 示例 |
| --- | --- | --- |
| Source0 | 非端口事件，需要手动唤醒 | performSelector、自定义事件 |
| Source1 | 基于 Mach Port，可由内核唤醒 | 系统事件、跨线程通信 |

Source0 不能主动唤醒 RunLoop，需要先标记再唤醒。Source1 可以通过 Mach Port 唤醒线程。

### 3.3 Timer

Timer 是基于时间的事件源，例如 `NSTimer`、`CFRunLoopTimer`。

```objc
NSTimer *timer = [NSTimer timerWithTimeInterval:1
                                         target:self
                                       selector:@selector(tick)
                                       userInfo:nil
                                        repeats:YES];
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

Timer 不是实时系统，触发时间会受到 RunLoop 忙碌、Mode、线程调度影响。

### 3.4 Observer

Observer 用于监听 RunLoop 状态变化。

常见状态：

| 状态 | 含义 |
| --- | --- |
| `kCFRunLoopEntry` | 进入 RunLoop |
| `kCFRunLoopBeforeTimers` | 即将处理 Timer |
| `kCFRunLoopBeforeSources` | 即将处理 Source |
| `kCFRunLoopBeforeWaiting` | 即将休眠 |
| `kCFRunLoopAfterWaiting` | 从休眠中醒来 |
| `kCFRunLoopExit` | 退出 RunLoop |

---

## 4. RunLoop 运行流程

简化流程：

```text
进入 RunLoop
  ↓
通知 Observers: Entry
  ↓
通知 Observers: BeforeTimers
  ↓
通知 Observers: BeforeSources
  ↓
处理 Blocks
  ↓
处理 Source0
  ↓
如果有 Source1 就处理
  ↓
通知 Observers: BeforeWaiting
  ↓
mach_msg 进入内核休眠
  ↓
事件到达，被唤醒
  ↓
通知 Observers: AfterWaiting
  ↓
处理 Timer / Source / Block
  ↓
根据结果退出或继续循环
```

核心点：

- 无事可做时线程会休眠，不消耗 CPU
- 事件到达后通过内核唤醒
- 每次运行只处理当前 Mode 下的事件源
- 主线程 RunLoop 与 UI 事件、Timer、渲染提交密切相关

---

## 5. 用户态与内核态

RunLoop 休眠依赖 `mach_msg`。

```text
用户态
  ↓ mach_msg
内核态等待消息
  ↓ 事件到达
内核唤醒线程
  ↓
回到用户态处理事件
```

这就是 RunLoop 能做到“没有事件时不占 CPU”的原因。

触摸事件链路简化：

```text
硬件触摸
  ↓
IOKit
  ↓
SpringBoard
  ↓
App Mach Port
  ↓
主线程 RunLoop 被唤醒
  ↓
UIApplication 分发事件
  ↓
UIWindow hit-test
  ↓
UIView / Gesture Recognizer
```

---

## 6. RunLoop 与 App 启动

主线程启动流程简化：

```text
main()
  ↓
UIApplicationMain
  ↓
创建 UIApplication
  ↓
创建 AppDelegate / SceneDelegate
  ↓
设置主 RunLoop
  ↓
进入事件循环
```

`UIApplicationMain` 内部会启动主线程 RunLoop，使 App 能持续接收触摸、Timer、系统事件、渲染提交等。

---

## 7. RunLoop 与 AutoreleasePool

主线程 AutoreleasePool 由系统通过 RunLoop Observer 管理：

```text
kCFRunLoopEntry
  ↓ 创建 AutoreleasePool

kCFRunLoopBeforeWaiting
  ↓ 释放旧 Pool
  ↓ 创建新 Pool

kCFRunLoopExit
  ↓ 释放 Pool
```

意义：

- 每轮事件处理结束后清理 autorelease 对象
- 避免临时对象长期堆积
- 子线程如果有大量临时对象，应手动添加 `@autoreleasepool`

```objc
dispatch_async(queue, ^{
    @autoreleasepool {
        [self processLargeFiles];
    }
});
```

---

## 8. RunLoop 与 GCD

GCD 和 RunLoop 不是同一个东西。

| 机制 | 职责 |
| --- | --- |
| RunLoop | 管理线程事件循环、休眠和唤醒 |
| GCD | 管理任务队列和线程池调度 |

主队列任务最终在主线程执行，主线程 RunLoop 会被唤醒处理这些任务。

```objc
dispatch_async(dispatch_get_main_queue(), ^{
    self.label.text = @"Done";
});
```

全局队列任务在线程池执行，通常不需要手动管理 RunLoop。

---

## 9. RunLoop 实际应用

### 9.1 常驻线程

适用于需要固定线程处理任务的场景，例如音频、Socket、长期串行任务。

```objc
- (void)run {
    @autoreleasepool {
        NSRunLoop *runLoop = [NSRunLoop currentRunLoop];
        [runLoop addPort:[NSMachPort port] forMode:NSDefaultRunLoopMode];

        while (!self.stopped) {
            [runLoop runMode:NSDefaultRunLoopMode beforeDate:[NSDate distantFuture]];
        }
    }
}
```

退出时应调用：

```objc
CFRunLoopStop(CFRunLoopGetCurrent());
```

不要只启动不退出，否则会泄漏线程资源。

### 9.2 Timer 滚动失效

问题：

```objc
[NSTimer scheduledTimerWithTimeInterval:1
                                 target:self
                               selector:@selector(tick)
                               userInfo:nil
                                repeats:YES];
```

默认加入 Default Mode。ScrollView 滚动时主 RunLoop 切到 Tracking Mode，Timer 不触发。

解决：

```objc
[[NSRunLoop mainRunLoop] addTimer:timer forMode:NSRunLoopCommonModes];
```

### 9.3 `performSelector:afterDelay:`

`performSelector:afterDelay:` 依赖当前线程 RunLoop。如果在没有运行 RunLoop 的子线程中调用，可能不会执行。

```objc
dispatch_async(dispatch_get_global_queue(0, 0), ^{
    [self performSelector:@selector(work) withObject:nil afterDelay:1];
});
```

更稳妥：

```objc
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, NSEC_PER_SEC),
               dispatch_get_global_queue(0, 0), ^{
    [self work];
});
```

### 9.4 卡顿监控

通过 Observer 监听主 RunLoop 状态，如果长时间停留在某个阶段，可认为主线程卡顿。

```text
BeforeSources
  ↓ 长时间不进入 BeforeWaiting
说明主线程可能被任务占用
```

工程上还需要结合：

- 主线程堆栈采样
- FPS
- MetricKit
- Time Profiler
- 页面维度统计

单靠 RunLoop 状态不能定位具体代码行。

### 9.5 CADisplayLink

`CADisplayLink` 与屏幕刷新同步，常用于动画和渲染驱动。

```objc
CADisplayLink *link = [CADisplayLink displayLinkWithTarget:self selector:@selector(render:)];
[link addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
```

它依赖 RunLoop，Mode 配置不当也会受滚动影响。

---

## 10. Timer 对比

| 类型 | 是否依赖 RunLoop | 适合场景 |
| --- | --- | --- |
| `NSTimer` | 是 | UI 相关普通定时 |
| `CADisplayLink` | 是 | 屏幕刷新同步 |
| GCD Timer | 否 | 后台定时、较高精度 |

GCD Timer 示例：

```objc
dispatch_source_t timer = dispatch_source_create(DISPATCH_SOURCE_TYPE_TIMER, 0, 0, queue);
dispatch_source_set_timer(timer,
                          dispatch_time(DISPATCH_TIME_NOW, 0),
                          NSEC_PER_SEC,
                          0.1 * NSEC_PER_SEC);
dispatch_source_set_event_handler(timer, ^{
    [self tick];
});
dispatch_resume(timer);
```

---

## 11. 面试高频问题

1. **RunLoop 是什么？**  
   RunLoop 是线程的事件循环，让线程在有事件时处理事件，无事件时休眠等待。

2. **主线程 RunLoop 什么时候启动？**  
   `UIApplicationMain` 内部启动主线程 RunLoop。

3. **子线程默认有 RunLoop 吗？**  
   可以懒加载创建，但不会自动运行；线程函数执行完仍会退出。

4. **为什么滚动时 Timer 不触发？**  
   滚动时 RunLoop 处于 `UITrackingRunLoopMode`，只加入 Default Mode 的 Timer 不会被处理。

5. **RunLoop 如何休眠和唤醒？**  
   通过 `mach_msg` 进入内核等待，Source1、Timer、GCD 主队列任务等事件到达后唤醒。

6. **RunLoop 和 AutoreleasePool 的关系？**  
   主线程通过 RunLoop Observer 在每轮循环创建和释放 AutoreleasePool。

---

## 12. 总结

RunLoop 是 iOS 线程模型和事件处理的核心。主线程依赖 RunLoop 处理触摸、Timer、GCD 主队列任务、渲染提交和 AutoreleasePool；子线程可以通过 RunLoop 实现常驻和事件驱动。理解 Mode、Source、Timer、Observer，是理解滚动、定时器、卡顿监控和主线程事件循环的关键。
