# iOS Runtime 底层原理

## 1. Runtime 概述

Runtime 通常指 Objective-C Runtime。它是一套运行时系统和 C API，支撑 Objective-C 的动态特性。

Runtime 负责：

- 类和对象管理
- 消息发送
- 方法查找和缓存
- 消息转发
- 分类加载
- 协议管理
- 关联对象
- Method Swizzling
- KVO 动态子类

核心头文件：

```objc
#import <objc/runtime.h>
#import <objc/message.h>
```

---

## 2. App 启动与类加载

### 2.1 main 之前发生什么

简化流程：

```text
dyld 加载 Mach-O
  ↓
加载依赖动态库
  ↓
绑定符号
  ↓
Runtime 读取类、分类、协议信息
  ↓
调用 +load
  ↓
执行 C++ 静态构造 / constructor
  ↓
进入 main
```

### 2.2 `+load`

特点：

- 类或分类加载进 Runtime 时调用
- 主类和分类的 `+load` 都会调用
- 调用时机早于 `main`
- 不依赖消息发送机制
- 不适合耗时操作

```objc
+ (void)load {
    NSLog(@"class loaded");
}
```

使用建议：

- 只做极少量注册
- 避免网络、I/O、数据库
- 避免复杂依赖
- Swizzling 要保证幂等

### 2.3 `+initialize`

特点：

- 类第一次收到消息前调用
- 通过消息发送触发
- 懒加载
- 父类可能先于子类调用

```objc
+ (void)initialize {
    if (self == [User class]) {
        // init once
    }
}
```

现代代码更推荐显式初始化或 `dispatch_once`。

---

## 3. 类与对象结构

### 3.1 objc_object

```c
struct objc_object {
    Class isa;
};
```

实例对象通过 `isa` 找到类对象。

### 3.2 objc_class

```c
struct objc_class {
    Class isa;
    Class superclass;
    cache_t cache;
    class_data_bits_t bits;
};
```

类对象保存：

- 父类关系
- 方法缓存
- 方法列表
- 属性列表
- 协议列表
- 成员变量描述

### 3.3 isa 优化

ARM64 中 `isa` 通常不是单纯指针，而是 non-pointer isa，利用位域存储额外信息：

- 类指针
- 引用计数相关 bit
- 是否有关联对象
- 是否有 C++ 析构
- 是否正在释放

这能减少 SideTable 访问，提高性能。

### 3.4 class_rw_t 与 class_ro_t

```text
class_ro_t
  ├── 编译期确定
  ├── 类名
  ├── ivar
  ├── method list
  └── protocol list

class_rw_t
  ├── 运行时可变
  ├── 合并 category
  ├── 动态添加方法
  └── 动态添加协议
```

Runtime 加载类后，会把编译期只读信息包装成运行时可变结构。

---

## 4. 消息发送

### 4.1 objc_msgSend

```objc
[obj doSomething];
```

底层：

```c
objc_msgSend(obj, @selector(doSomething));
```

方法默认隐含两个参数：

- `self`
- `_cmd`

### 4.2 完整流程

```text
objc_msgSend
  ↓
receiver == nil ?
  ↓
从 isa 获取 Class
  ↓
查 cache_t
  ↓ 命中
调用 IMP
  ↓ 未命中
查 method list
  ↓ 未命中
沿 superclass 查找
  ↓ 未命中
消息转发
```

### 4.3 为什么用汇编

`objc_msgSend` 是极高频路径，需要：

- 快速访问寄存器
- 快速判断 nil
- 快速查 cache
- 尾调用跳转 IMP
- 避免额外栈开销

因此 Runtime 使用汇编实现关键路径。

### 4.4 直接调用风险

不推荐业务代码直接调用 `objc_msgSend`。如果必须调用，需要按真实函数签名强转：

```objc
void (*send)(id, SEL, NSString *) = (void *)objc_msgSend;
send(obj, @selector(setName:), @"Tom");
```

返回结构体、浮点值时还要关注 ABI 差异。

---

## 5. 方法缓存

`cache_t` 保存 `SEL -> IMP` 映射。

```text
第一次调用:
  查方法列表和父类链
  ↓
  缓存 IMP

后续调用:
  cache 命中
  ↓
  直接跳转
```

Swizzling 后 Runtime 会处理缓存一致性，但业务上仍要避免在复杂时机频繁交换方法。

---

## 6. 消息转发

### 6.1 三阶段

```text
1. 动态方法解析
   +resolveInstanceMethod:
   +resolveClassMethod:

2. 快速转发
   -forwardingTargetForSelector:

3. 慢速转发
   -methodSignatureForSelector:
   -forwardInvocation:
```

### 6.2 动态添加方法

```objc
void runIMP(id self, SEL _cmd) {
    NSLog(@"run");
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(run)) {
        class_addMethod(self, sel, (IMP)runIMP, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

### 6.3 快速转发

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([self.proxy respondsToSelector:aSelector]) {
        return self.proxy;
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

### 6.4 慢速转发

```objc
- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector {
    return [self.target methodSignatureForSelector:aSelector];
}

- (void)forwardInvocation:(NSInvocation *)invocation {
    [invocation invokeWithTarget:self.target];
}
```

慢速转发适合代理、多播、AOP，但性能低且可读性差。

---

## 7. Category

### 7.1 Category 数据

Category 编译后包含：

- 分类名
- 目标类名
- 实例方法列表
- 类方法列表
- 协议列表
- 属性列表

### 7.2 加载过程

```text
Runtime 读取 Category
  ↓
找到目标类
  ↓
把方法列表附加到 class_rw_t
  ↓
清理/更新方法缓存
```

分类方法可能覆盖主类同名方法，但不应依赖覆盖顺序。

### 7.3 关联对象

Category 不能直接添加 ivar，可以使用关联对象。

```objc
static void *NameKey = &NameKey;

- (void)setIoser_name:(NSString *)name {
    objc_setAssociatedObject(self, NameKey, name, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)ioser_name {
    return objc_getAssociatedObject(self, NameKey);
}
```

key 推荐使用静态变量地址，避免冲突。

---

## 8. Method Swizzling

### 8.1 原理

Swizzling 本质是交换两个 Method 的 IMP。

```objc
Method original = class_getInstanceMethod(cls, @selector(viewDidAppear:));
Method swizzled = class_getInstanceMethod(cls, @selector(ioser_viewDidAppear:));
method_exchangeImplementations(original, swizzled);
```

### 8.2 安全模板

```objc
static inline void SwizzleInstanceMethod(Class cls, SEL originalSEL, SEL swizzledSEL) {
    Method originalMethod = class_getInstanceMethod(cls, originalSEL);
    Method swizzledMethod = class_getInstanceMethod(cls, swizzledSEL);

    if (!originalMethod || !swizzledMethod) {
        return;
    }

    BOOL didAdd = class_addMethod(
        cls,
        originalSEL,
        method_getImplementation(swizzledMethod),
        method_getTypeEncoding(swizzledMethod)
    );

    if (didAdd) {
        class_replaceMethod(
            cls,
            swizzledSEL,
            method_getImplementation(originalMethod),
            method_getTypeEncoding(originalMethod)
        );
    } else {
        method_exchangeImplementations(originalMethod, swizzledMethod);
    }
}
```

为什么先 `class_addMethod`：如果原方法只存在于父类，直接交换会影响父类。先添加可把影响限制在当前类。

### 8.3 使用边界

适合：

- 埋点
- 调试工具
- 兼容性修复
- AOP

风险：

- 多个库交换同一方法顺序不确定
- 影响系统类稳定性
- Debug 困难
- 容易引入递归调用

---

## 9. KVO Runtime 原理

KVO 通过动态子类和 isa-swizzling 实现。

```text
Person 对象
  ↓ addObserver
Runtime 创建 NSKVONotifying_Person
  ↓
对象 isa 指向动态子类
  ↓
动态子类重写 setter
  ↓
setter 前后发送通知
```

动态子类会重写：

- 被观察属性 setter
- `class`
- `dealloc`
- `_isKVOA`

重写 `class` 是为了隐藏真实动态子类，让外部仍看到原类。

---

## 10. Runtime 实际应用

### 10.1 字典转模型

```objc
+ (instancetype)modelWithDictionary:(NSDictionary *)dictionary {
    id model = [[self alloc] init];

    Class cls = self;
    while (cls && cls != [NSObject class]) {
        unsigned int count = 0;
        objc_property_t *properties = class_copyPropertyList(cls, &count);

        for (unsigned int i = 0; i < count; i++) {
            const char *name = property_getName(properties[i]);
            NSString *key = [NSString stringWithUTF8String:name];
            id value = dictionary[key];

            if (value && value != [NSNull null]) {
                [model setValue:value forKey:key];
            }
        }

        free(properties);
        cls = class_getSuperclass(cls);
    }

    return model;
}
```

真实项目还要处理字段映射、嵌套对象、数组类型、类型转换、只读属性等。

### 10.2 埋点

通过 Swizzling `viewDidAppear:` 可以统一页面曝光，但要注意：

- 页面命名规则
- 多次曝光
- 容器控制器
- SwiftUI 页面
- 性能和稳定性

### 10.3 防 Crash

Runtime 可拦截部分崩溃，但不建议静默吞错。合理策略：

- Debug 保持崩溃
- Release 有限兜底
- 必须上报堆栈和上下文
- 核心业务不吞错

---

## 11. Runtime API 清单

| API | 作用 |
| --- | --- |
| `object_getClass` | 获取对象 isa 指向 |
| `objc_getClass` | 根据类名获取类 |
| `class_getSuperclass` | 获取父类 |
| `class_copyMethodList` | 获取方法列表 |
| `class_copyPropertyList` | 获取属性列表 |
| `class_addMethod` | 添加方法 |
| `class_replaceMethod` | 替换方法 |
| `method_exchangeImplementations` | 交换实现 |
| `method_getImplementation` | 获取 IMP |
| `objc_setAssociatedObject` | 设置关联对象 |
| `objc_getAssociatedObject` | 获取关联对象 |

---

## 12. 面试高频问题

1. **Runtime 是什么？**  
   Runtime 是 Objective-C 的运行时系统，支持动态类型、消息发送、方法查找、消息转发、关联对象等能力。

2. **消息发送流程？**  
   判空，查 `isa`，查 cache，查方法列表，沿父类链查找，找不到进入消息转发。

3. **`+load` 和 `+initialize` 区别？**  
   `+load` 类加载时调用，早于 main；`+initialize` 首次收到消息前懒调用。

4. **Category 为什么不能添加 ivar？**  
   对象内存布局在编译期已确定，运行时不能改变已有对象大小。

5. **关联对象存在哪里？**  
   Runtime 维护的全局关联表中，不在对象本体内。

6. **KVO 原理？**  
   Runtime 动态创建子类，修改对象 `isa`，重写 setter 发送通知。

7. **Swizzling 风险？**  
   调用顺序不确定、影响系统类、递归、与三方库冲突、调试困难。

---

## 13. 总结

Runtime 是 Objective-C 动态能力的基础。消息发送、方法缓存、消息转发、Category、关联对象、Swizzling、KVO 都依赖 Runtime。工程中要清楚 Runtime 能力强但风险也高，应优先使用静态、清晰的设计，只在确有收益时使用动态能力。
