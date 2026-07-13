# Objective-C 底层原理

## 1. Objective-C 的本质

Objective-C 是 C 语言的超集，在 C 的基础上加入了 Smalltalk 风格的动态消息机制。它的核心不是“调用函数”，而是“给对象发送消息”。

```objc
[person sayHello:@"Tom"];
```

编译后可理解为：

```c
objc_msgSend(person, @selector(sayHello:), @"Tom");
```

区别在于：

| 维度 | C | Objective-C |
| --- | --- | --- |
| 调用方式 | 函数调用 | 消息发送 |
| 绑定时机 | 编译期静态绑定 | 运行时动态查找 |
| 类型系统 | 结构体、函数 | 对象、类、元类、协议、分类 |
| 扩展能力 | 编译期决定 | Runtime 可动态添加方法、交换实现、关联对象 |

Objective-C 的动态性主要来自 Runtime。类、对象、方法、协议、分类等信息都会被编译进 Mach-O，并在运行时由 Runtime 加载和管理。

---

## 2. 对象、类、元类

### 2.1 对象的本质

任意 Objective-C 对象底层都至少包含一个 `isa` 指针：

```c
struct objc_object {
    Class isa;
};
```

对象实例保存成员变量，方法实现不在对象中，而在类对象的方法列表中。

```text
person 实例
  ├── isa -> Person 类对象
  ├── name
  └── age

Person 类对象
  ├── superclass -> NSObject
  ├── cache
  ├── method list
  ├── property list
  └── protocol list
```

### 2.2 类对象

类对象描述实例对象的结构和行为，底层核心结构可简化为：

```c
struct objc_class {
    Class isa;
    Class superclass;
    cache_t cache;
    class_data_bits_t bits;
};
```

关键字段：

- `isa`：类对象的 `isa` 指向元类
- `superclass`：父类指针
- `cache`：方法缓存，加速消息发送
- `bits`：指向类的运行时数据，例如方法、属性、协议等

### 2.3 元类

元类保存类方法。实例方法存在类对象中，类方法存在元类对象中。

```text
实例对象 --isa--> 类对象 --isa--> 元类对象 --isa--> 根元类
   |               |               |
实例变量          实例方法         类方法
```

`[Person alloc]` 是给 `Person` 类对象发送 `alloc` 消息，方法查找会从 `Person` 的元类开始。

### 2.4 isa 走位

```text
person 实例
  isa -> Person 类对象
          isa -> Person 元类
                  isa -> NSObject 元类
                          isa -> NSObject 元类自身

Person 类对象
  superclass -> NSObject 类对象

Person 元类
  superclass -> NSObject 元类
```

常见结论：

- 实例方法从类对象查找
- 类方法从元类对象查找
- 类对象本身也是对象
- 元类最终也会走到根元类

### 2.5 `self` 与 `super`

`self` 是当前消息接收者。`super` 不是对象，而是编译器指令，表示从父类开始查找方法，但消息接收者仍然是 `self`。

```objc
- (void)viewDidLoad {
    [super viewDidLoad];
}
```

可理解为：

```c
struct objc_super arg = {
    .receiver = self,
    .super_class = class_getSuperclass(object_getClass(self))
};
objc_msgSendSuper(&arg, @selector(viewDidLoad));
```

所以 `[super class]` 返回的通常仍是当前对象的真实类。

---

## 3. 方法与消息发送

### 3.1 SEL、IMP、Method

| 概念 | 含义 |
| --- | --- |
| `SEL` | 方法名的唯一标识，本质可理解为运行时注册的字符串 |
| `IMP` | 方法实现的函数指针 |
| `Method` | 方法描述，包含 `SEL`、`IMP`、类型编码 |

```c
typedef struct method_t *Method;
typedef struct objc_selector *SEL;
typedef id (*IMP)(id, SEL, ...);
```

一个 Objective-C 方法至少隐含两个参数：

```objc
- (void)say:(NSString *)text;
```

底层类似：

```c
void say(id self, SEL _cmd, NSString *text);
```

### 3.2 消息发送流程

```text
objc_msgSend(receiver, selector)
  ↓
receiver 是否为 nil
  ↓
从 receiver->isa 找到类对象
  ↓
查找 cache_t
  ↓ 命中
直接调用 IMP
  ↓ 未命中
查找当前类 method list
  ↓ 未命中
沿 superclass 向上查找
  ↓ 未命中
进入消息转发
```

给 `nil` 发送消息不会崩溃，返回值通常为 0、nil 或对应零值。这是 Objective-C 的语言特性，但不能滥用，否则会隐藏问题。

### 3.3 方法缓存

Runtime 使用 `cache_t` 缓存已经查找过的方法，实现快速分发。

```text
第一次调用:
  method list / superclass chain 查找
  ↓
  缓存 SEL -> IMP

后续调用:
  cache 命中
  ↓
  直接跳转 IMP
```

方法缓存是 `objc_msgSend` 性能接近普通虚函数调用的重要原因。

### 3.4 消息转发三阶段

当方法查找失败后，Runtime 会进入消息转发：

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

动态添加方法示例：

```objc
void dynamicMethod(id self, SEL _cmd) {
    NSLog(@"dynamic method");
}

+ (BOOL)resolveInstanceMethod:(SEL)sel {
    if (sel == @selector(run)) {
        class_addMethod(self, sel, (IMP)dynamicMethod, "v@:");
        return YES;
    }
    return [super resolveInstanceMethod:sel];
}
```

快速转发示例：

```objc
- (id)forwardingTargetForSelector:(SEL)aSelector {
    if ([self.helper respondsToSelector:aSelector]) {
        return self.helper;
    }
    return [super forwardingTargetForSelector:aSelector];
}
```

慢速转发可以拿到完整 `NSInvocation`，适合代理、多播、AOP 等场景，但性能更低。

---

## 4. 属性、成员变量与内存语义

### 4.1 `@property` 做了什么

```objc
@property (nonatomic, copy) NSString *name;
```

通常会生成：

- 成员变量 `_name`
- getter：`name`
- setter：`setName:`
- 属性元数据

属性修饰符决定 setter/getter 的内存和线程语义。

### 4.2 常见修饰符

| 修饰符 | 含义 | 场景 |
| --- | --- | --- |
| `strong` | 强引用持有对象 | 普通对象 |
| `weak` | 弱引用，释放后自动置 nil | delegate、反向引用 |
| `copy` | 复制对象 | `NSString`、集合、Block |
| `assign` | 直接赋值 | 基础类型 |
| `unsafe_unretained` | 不持有且不置 nil | 极少使用 |
| `nonatomic` | 非原子访问 | iOS 常用 |
| `atomic` | getter/setter 原子性 | 很少使用 |

`atomic` 不是线程安全方案，它只保证单次属性读写的原子性，不保护复合操作。

### 4.3 为什么 `NSString` 用 `copy`

```objc
@property (nonatomic, copy) NSString *name;

NSMutableString *text = [NSMutableString stringWithString:@"A"];
self.name = text;
[text appendString:@"B"];
```

如果使用 `strong`，属性可能受到外部可变字符串影响。`copy` 可以保存赋值时的不可变快照。

### 4.4 Block 属性为什么用 `copy`

```objc
@property (nonatomic, copy) void (^completion)(void);
```

Block 可能在栈上创建，`copy` 后移动到堆上，才能在作用域结束后安全持有。ARC 会自动处理很多场景，但属性语义仍应写 `copy`。

---

## 5. Category 与 Extension

### 5.1 Category 的作用

Category 可以在不修改原类源码的情况下添加方法：

```objc
@interface NSString (Trim)
- (NSString *)ioser_trimmedString;
@end
```

Category 编译后会生成分类结构体，包含：

- 分类名
- 目标类名
- 实例方法列表
- 类方法列表
- 协议列表
- 属性列表

### 5.2 Category 加载过程

Runtime 加载分类后，会把分类中的方法、协议、属性附加到主类上。

```text
dyld 加载 Mach-O
  ↓
Runtime 读取类和分类
  ↓
把 Category 方法列表合并到目标类
  ↓
方法查找时可命中分类方法
```

分类方法通常会插入到方法列表前面，所以分类方法可能覆盖主类同名方法。多个分类存在同名方法时，结果受编译链接顺序影响，不应依赖。

### 5.3 Category 为什么不能直接加 ivar

对象大小在类编译完成后已经确定，运行时不能随意改变已有对象布局。Category 不能直接添加实例变量。

如果需要在分类中保存状态，可以使用关联对象：

```objc
static void *TitleKey = &TitleKey;

- (void)setIoser_title:(NSString *)title {
    objc_setAssociatedObject(self, TitleKey, title, OBJC_ASSOCIATION_COPY_NONATOMIC);
}

- (NSString *)ioser_title {
    return objc_getAssociatedObject(self, TitleKey);
}
```

关联对象不是真正的 ivar，而是 Runtime 维护的全局关联表。

### 5.4 Class Extension

Class Extension 写在 `.m` 文件中，常用于声明私有属性和私有方法：

```objc
@interface UserViewController ()
@property (nonatomic, strong) UserService *service;
@end
```

Extension 在编译期参与类定义，可以添加 ivar；Category 是运行时附加，不能直接添加 ivar。

---

## 6. KVC 与 KVO

### 6.1 KVC 查找规则

KVC 允许通过字符串访问属性：

```objc
[person setValue:@"Tom" forKey:@"name"];
NSString *name = [person valueForKey:@"name"];
```

`setValue:forKey:` 大致查找顺序：

```text
setName:
_setName:
accessInstanceVariablesDirectly == YES
  ↓
_name
_isName
name
isName
  ↓
setValue:forUndefinedKey:
```

KVC 风险：

- key 写错运行时才暴露
- 类型不匹配可能崩溃
- 可绕过 setter 直接访问 ivar
- 对集合和嵌套 key path 要注意异常处理

### 6.2 KVO 实现原理

KVO 基于 Runtime 动态子类实现：

```text
原对象: Person
  ↓ addObserver
动态创建: NSKVONotifying_Person
  ↓
修改对象 isa 指向动态子类
  ↓
重写 setter，在 setter 前后发送通知
```

动态子类通常会重写：

- 被观察属性的 setter
- `class`
- `dealloc`
- `_isKVOA`

### 6.3 KVO 注意点

- 旧式 KVO 需要成对添加和移除观察者
- key path 是字符串，缺少编译期检查
- 手动修改 ivar 不一定触发 KVO
- 可用 `willChangeValueForKey:` 和 `didChangeValueForKey:` 手动触发
- 现代 Swift 更推荐 Combine、Observation 或明确回调

---

## 7. Block 底层原理

### 7.1 Block 的本质

Block 本质是结构体，包含函数指针和捕获变量。

```c
struct Block_layout {
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
};
```

### 7.2 Block 类型

| 类型 | 位置 | 特点 |
| --- | --- | --- |
| Global Block | 全局区 | 不捕获自动变量 |
| Stack Block | 栈 | 捕获自动变量，作用域结束失效 |
| Malloc Block | 堆 | Stack Block copy 后 |

```objc
void (^globalBlock)(void) = ^{
    NSLog(@"global");
};

int age = 10;
void (^stackOrMallocBlock)(void) = ^{
    NSLog(@"%d", age);
};
```

ARC 下，赋值给强引用变量或作为属性持有时，编译器通常会自动 copy。

### 7.3 变量捕获

| 变量类型 | 捕获方式 |
| --- | --- |
| 局部自动变量 | 值捕获 |
| 静态局部变量 | 指针捕获 |
| 全局变量 | 直接访问 |
| 对象变量 | 强引用捕获，除非使用 `__weak` |
| `__block` 变量 | 包装成 byref 结构体 |

`__block` 允许在 Block 内修改变量：

```objc
__block NSInteger count = 0;
void (^block)(void) = ^{
    count += 1;
};
```

### 7.4 Block 循环引用

```objc
self.completion = ^{
    [self reloadData];
};
```

如果 `self` 强引用 `completion`，Block 又强引用 `self`，会循环引用。

常见写法：

```objc
__weak typeof(self) weakSelf = self;
self.completion = ^{
    __strong typeof(weakSelf) self = weakSelf;
    if (!self) return;
    [self reloadData];
};
```

---

## 8. ARC 与内存管理

### 8.1 ARC 的本质

ARC 是编译器特性，不是垃圾回收。编译器在合适位置插入 `retain`、`release`、`autorelease`。

```objc
Person *p = [[Person alloc] init];
```

ARC 会根据所有权规则自动管理对象生命周期。

### 8.2 引用计数存储

对象引用计数可能存储在：

- `isa` 位域中
- SideTable 的引用计数表中

当内联引用计数放不下或对象需要弱引用表等额外信息时，会使用 SideTable。

### 8.3 weak 实现

weak 的关键是弱引用表：

```text
weak 指针赋值
  ↓
objc_storeWeak
  ↓
在 weak_table 中记录：对象地址 -> 所有 weak 指针地址

对象 dealloc
  ↓
查找 weak_table
  ↓
遍历所有 weak 指针并置 nil
```

所以 weak 可以自动置 nil，而 `unsafe_unretained` 不会。

### 8.4 AutoreleasePool

AutoreleasePool 用于延迟释放对象。主线程 RunLoop 会自动管理池：

```text
RunLoop Entry -> 创建 pool
BeforeWaiting -> 释放并重建 pool
RunLoop Exit -> 释放 pool
```

大量临时对象循环中应手动添加：

```objc
for (NSInteger i = 0; i < 100000; i++) {
    @autoreleasepool {
        NSString *value = [NSString stringWithFormat:@"%ld", (long)i];
        [self process:value];
    }
}
```

---

## 9. Runtime 常用 API

| API | 用途 |
| --- | --- |
| `object_getClass` | 获取对象的 isa 指向 |
| `class_getSuperclass` | 获取父类 |
| `class_copyMethodList` | 获取方法列表 |
| `class_copyPropertyList` | 获取属性列表 |
| `class_addMethod` | 动态添加方法 |
| `method_exchangeImplementations` | 交换方法实现 |
| `objc_setAssociatedObject` | 设置关联对象 |
| `objc_getAssociatedObject` | 获取关联对象 |

示例：遍历属性：

```objc
unsigned int count = 0;
objc_property_t *properties = class_copyPropertyList([User class], &count);

for (unsigned int i = 0; i < count; i++) {
    const char *name = property_getName(properties[i]);
    NSLog(@"%s", name);
}

free(properties);
```

---

## 10. 工程实践与风险边界

### 10.1 Method Swizzling

安全模板：

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

注意：

- 必须保证只交换一次
- 不要大范围交换系统核心类
- 交换前确认方法存在
- 保留原实现调用链
- Release 中启用要有监控和灰度

### 10.2 Runtime 防 Crash 的边界

通过 Runtime 拦截数组越界、字典插入 nil、未识别 selector 可以降低崩溃率，但风险很高：

- 可能掩盖真实 bug
- 改变系统类行为
- 与三方 SDK 冲突
- 数据错误但不崩溃，问题更难定位

更合理的策略：

- Debug 环境保持崩溃
- Release 只对白名单问题做兜底
- 兜底后必须上报堆栈和上下文
- 核心业务不要静默吞错

### 10.3 `+load` 使用边界

`+load` 会影响启动耗时。适合做极少量注册，不适合做：

- 网络请求
- 数据库初始化
- 大量 I/O
- 复杂依赖组装
- 业务状态判断

---

## 11. 面试高频问题

1. **Objective-C 为什么是动态语言？**  
   因为方法查找、类型判断、消息转发、方法添加、实现交换等大量行为可以在运行时完成。

2. **`objc_msgSend` 流程是什么？**  
   先判空，再从 `isa` 找类，查 cache，未命中查方法列表，再沿父类链查找，仍找不到进入消息转发。

3. **Category 和 Extension 区别？**  
   Extension 是编译期类扩展，可加 ivar；Category 是运行时附加，不能直接加 ivar。

4. **weak 为什么自动置 nil？**  
   Runtime 通过 weak table 记录所有弱引用地址，对象释放时遍历并置 nil。

5. **KVO 的原理是什么？**  
   Runtime 动态创建子类，重写 setter，并把对象 `isa` 指向该子类，在 setter 前后发送变更通知。

6. **Block 为什么会循环引用？**  
   对象强持有 Block，Block 默认强捕获对象，就形成环。可用 `weak/strong` 模式打破。

---

## 12. 总结

Objective-C 的核心是 Runtime 驱动的动态对象系统。对象通过 `isa` 找到类，类通过方法列表和缓存响应消息，找不到方法时进入消息转发。Category、KVO、关联对象、Swizzling、Block、ARC 都建立在这套运行时和内存管理机制之上。

工程上应理解这些机制的边界：Runtime 能增强灵活性，也会降低可读性和稳定性。能用静态设计解决的问题，不要优先依赖 Runtime 黑魔法。
