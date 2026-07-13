# LLVM 深入介绍与学习路线

## 1. LLVM 是什么

LLVM 最初是 Low Level Virtual Machine 的缩写，但现在 LLVM 已经不再只是一个“虚拟机”项目，而是一套模块化、可复用的编译器基础设施。

它提供了从中间表示、优化器、代码生成器、链接器、调试信息、运行时工具到静态分析工具的一整套能力。现代语言、编译器、构建系统、性能分析工具和安全工具都可能直接或间接使用 LLVM。

常见使用 LLVM 的技术：

- Clang：C、C++、Objective-C、Objective-C++ 编译器前端
- Swift 编译器：Swift 前端生成 SIL，进一步转换到 LLVM IR
- Rust 编译器：前端使用 MIR，后端通常使用 LLVM
- Julia：使用 LLVM 做 JIT 和代码生成
- Kotlin/Native：使用 LLVM 后端
- Emscripten：通过 LLVM 生成 WebAssembly
- Xcode 工具链：Clang、Swift、LLDB、LTO 等都与 LLVM 生态相关

LLVM 的核心价值是把编译器拆成可复用的模块：

```text
源代码
  ↓
前端 Frontend
  ↓
LLVM IR
  ↓
优化器 Optimizer
  ↓
后端 Backend
  ↓
目标机器码 / 汇编 / 对象文件
```

前端负责理解语言，后端负责理解机器，中间的 LLVM IR 负责把两者连接起来。

---

## 2. 为什么要学习 LLVM

学习 LLVM 不只是为了写编译器。对 iOS、Swift、Objective-C、C/C++、性能优化、安全分析、逆向工程、静态分析和工具链开发都有帮助。

### 2.1 对 iOS 开发的价值

LLVM 与 Apple 平台关系非常紧密：

- Xcode 使用 Clang 编译 C、C++、Objective-C
- Swift 编译器会把 Swift 中间表示转换为 LLVM IR
- `-O`、`-Onone`、`-Osize` 等优化选项最终会影响 LLVM 优化和代码生成
- Bitcode 曾经基于 LLVM IR 思路，用于 App Store 侧重编译优化
- LLDB 是 LLVM 项目的一部分
- Sanitizer 工具链基于 LLVM 插桩能力
- 静态分析器、代码检查工具、语法重写工具常常基于 Clang/LLVM

理解 LLVM 后，你能更清楚地回答这些问题：

- 为什么 Debug 和 Release 行为、性能、栈信息不同？
- 编译器如何内联函数？
- 为什么某些 Swift 代码会被优化到几乎没有运行时成本？
- ARC retain/release 是如何被插入和优化掉的？
- C++ 模板、Objective-C 消息发送、Swift 泛型最后如何变成机器指令？
- Address Sanitizer、Thread Sanitizer 为什么能发现内存和线程问题？

### 2.2 对编译原理学习的价值

传统编译原理教材经常停留在词法分析、语法分析、语义分析和简单代码生成。LLVM 提供了真实工业级工程，可以把理论落到实践中。

你可以通过 LLVM 学到：

- 编译器前端与后端如何解耦
- 中间表示为什么重要
- SSA 形式如何表达程序数据流
- 优化 Pass 如何组织
- 寄存器分配如何影响性能
- 目标平台 ABI 如何影响函数调用
- 调试信息和源代码位置如何保留

---

## 3. LLVM 的整体架构

### 3.1 前端 Frontend

前端负责把源语言转换成 LLVM IR。不同语言有不同的前端。

```text
C / C++ / Objective-C  -> Clang -> LLVM IR
Swift                 -> Swift Frontend -> SIL -> LLVM IR
Rust                  -> rustc Frontend -> MIR -> LLVM IR
```

前端通常负责：

- 词法分析：把字符流切分为 Token
- 语法分析：把 Token 组织成 AST
- 语义分析：类型检查、作用域检查、名字查找
- 语言特有转换：泛型、闭包、宏、属性、协议、异常等
- 生成 LLVM IR 或自己的高级中间表示

Clang 是 LLVM 官方生态中最典型的前端。

### 3.2 LLVM IR

LLVM IR 是 LLVM 的核心中间表示。它足够底层，可以接近机器；又足够抽象，可以独立于具体 CPU。

LLVM IR 有三种常见形态：

| 形态 | 后缀 | 说明 |
| --- | --- | --- |
| 文本 IR | `.ll` | 可读性较好，适合学习和调试 |
| Bitcode | `.bc` | 二进制编码，适合工具链传递 |
| 内存对象 | 无固定后缀 | LLVM 内部 C++ 对象模型 |

示例 C 代码：

```c
int add(int a, int b) {
    return a + b;
}
```

可能生成的 LLVM IR：

```llvm
define i32 @add(i32 %a, i32 %b) {
entry:
  %sum = add nsw i32 %a, %b
  ret i32 %sum
}
```

这里的含义：

- `define i32 @add`：定义一个返回 `i32` 的函数
- `i32 %a`、`i32 %b`：两个 32 位整数参数
- `%sum = add nsw i32 %a, %b`：执行有符号整数加法
- `ret i32 %sum`：返回结果

### 3.3 优化器 Optimizer

LLVM 优化器由一组 Pass 组成。每个 Pass 做一类优化或分析。

常见优化：

- Constant Folding：常量折叠
- Dead Code Elimination：删除无用代码
- Function Inlining：函数内联
- Loop Unrolling：循环展开
- Loop Vectorization：循环向量化
- Global Value Numbering：公共表达式消除
- Scalar Replacement of Aggregates：聚合类型标量替换
- Tail Call Optimization：尾调用优化

优化器的输入和输出通常都是 LLVM IR：

```text
未优化 LLVM IR
  ↓ opt
优化 Pass 管线
  ↓
优化后 LLVM IR
```

### 3.4 后端 Backend

后端负责把 LLVM IR 转换为目标平台机器代码。

后端主要阶段：

```text
LLVM IR
  ↓ Instruction Selection
机器相关中间表示
  ↓ Register Allocation
寄存器分配
  ↓ Instruction Scheduling
指令调度
  ↓ Code Emission
汇编 / 对象文件
```

目标平台包括：

- x86 / x86_64
- ARM / AArch64
- RISC-V
- WebAssembly
- PowerPC
- GPU 相关目标

在 Apple Silicon 上，最终常见目标是 AArch64。

---

## 4. LLVM IR 基础

### 4.1 LLVM IR 的核心特点

LLVM IR 有几个重要特点：

- 强类型
- 静态单赋值 SSA
- 显式控制流
- 显式内存访问
- 接近底层机器模型
- 独立于具体 CPU

#### 强类型

LLVM IR 中每个值都有类型：

```llvm
i1      ; 1 位布尔值
i8      ; 8 位整数
i32     ; 32 位整数
i64     ; 64 位整数
float   ; 32 位浮点
double  ; 64 位浮点
ptr     ; 指针
```

#### SSA 形式

SSA 是 Static Single Assignment，静态单赋值。每个虚拟寄存器只被赋值一次。

```llvm
%x = add i32 %a, %b
%y = mul i32 %x, 2
```

不能这样写：

```llvm
%x = add i32 %a, %b
%x = mul i32 %x, 2
```

SSA 让数据流更清晰，方便优化器分析。

### 4.2 Module、Function、BasicBlock、Instruction

LLVM IR 的层级可以理解为：

```text
Module
├── Global Variable
├── Function
│   ├── BasicBlock
│   │   ├── Instruction
│   │   └── Instruction
│   └── BasicBlock
└── Function
```

| 概念 | 说明 |
| --- | --- |
| Module | 一个编译单元，类似 `.c` 文件编译后的 IR 容器 |
| Function | 函数 |
| BasicBlock | 基本块，只能从入口进入，以终结指令结束 |
| Instruction | 一条 IR 指令 |

基本块通常以这些指令结束：

- `ret`
- `br`
- `switch`
- `invoke`
- `unreachable`

### 4.3 条件分支示例

C 代码：

```c
int max_value(int a, int b) {
    if (a > b) {
        return a;
    }
    return b;
}
```

未优化 IR 可能类似：

```llvm
define i32 @max_value(i32 %a, i32 %b) {
entry:
  %cmp = icmp sgt i32 %a, %b
  br i1 %cmp, label %then, label %else

then:
  ret i32 %a

else:
  ret i32 %b
}
```

优化后可能变成：

```llvm
define i32 @max_value(i32 %a, i32 %b) {
entry:
  %cmp = icmp sgt i32 %a, %b
  %result = select i1 %cmp, i32 %a, i32 %b
  ret i32 %result
}
```

`select` 类似三目运算符，可以减少分支。

### 4.4 循环示例

C 代码：

```c
int sum_to_n(int n) {
    int sum = 0;
    for (int i = 0; i <= n; i++) {
        sum += i;
    }
    return sum;
}
```

简化后的 LLVM IR：

```llvm
define i32 @sum_to_n(i32 %n) {
entry:
  br label %loop

loop:
  %i = phi i32 [ 0, %entry ], [ %next_i, %loop ]
  %sum = phi i32 [ 0, %entry ], [ %next_sum, %loop ]
  %cond = icmp sle i32 %i, %n
  br i1 %cond, label %body, label %exit

body:
  %next_sum = add nsw i32 %sum, %i
  %next_i = add nsw i32 %i, 1
  br label %loop

exit:
  ret i32 %sum
}
```

这个例子里最重要的是 `phi`。

### 4.5 PHI 节点

`phi` 用于在 SSA 形式中表达“来自不同控制流路径的值”。

```llvm
%x = phi i32 [ 0, %entry ], [ %next_x, %loop ]
```

含义是：

- 如果从 `%entry` 来，`%x` 的值是 `0`
- 如果从 `%loop` 来，`%x` 的值是 `%next_x`

在源代码中，循环变量会被多次赋值；在 SSA 中，每个变量只能赋值一次，所以需要 `phi` 表示不同路径汇合后的值。

### 4.6 栈内存：alloca、load、store

LLVM IR 中局部变量通常可以用 `alloca` 表示栈空间。

C 代码：

```c
int foo() {
    int x = 10;
    x = x + 1;
    return x;
}
```

未优化 IR 可能类似：

```llvm
define i32 @foo() {
entry:
  %x = alloca i32
  store i32 10, ptr %x
  %v = load i32, ptr %x
  %add = add nsw i32 %v, 1
  store i32 %add, ptr %x
  %result = load i32, ptr %x
  ret i32 %result
}
```

优化后可能变成：

```llvm
define i32 @foo() {
entry:
  ret i32 11
}
```

这展示了两个优化：

- Mem2Reg：把内存变量提升为 SSA 寄存器值
- Constant Folding：把 `10 + 1` 直接折叠为 `11`

---

## 5. Clang 与 LLVM

### 5.1 Clang 在 LLVM 体系中的位置

Clang 是 C 家族语言前端。它负责把 C、C++、Objective-C、Objective-C++ 转成 LLVM IR。

```text
source.c
  ↓ clang
AST
  ↓ CodeGen
LLVM IR
  ↓ LLVM Optimizer
Machine Code
```

Clang 的重要能力：

- 编译 C/C++/Objective-C
- 生成 AST
- 生成 LLVM IR
- 静态分析
- 格式化代码：clang-format
- 代码重写和迁移：clang-tidy、libTooling
- 诊断警告和错误

### 5.2 查看 AST

示例代码：

```c
int add(int a, int b) {
    return a + b;
}
```

命令：

```bash
clang -Xclang -ast-dump -fsyntax-only add.c
```

你会看到函数声明、参数声明、返回语句、二元运算表达式等 AST 节点。

### 5.3 生成 LLVM IR

生成文本 IR：

```bash
clang -S -emit-llvm add.c -o add.ll
```

生成 Bitcode：

```bash
clang -c -emit-llvm add.c -o add.bc
```

查看优化前 IR：

```bash
clang -S -emit-llvm -O0 add.c -o add_O0.ll
```

查看优化后 IR：

```bash
clang -S -emit-llvm -O2 add.c -o add_O2.ll
```

对比 `-O0` 和 `-O2` 是学习 LLVM 最有效的方法之一。

### 5.4 生成汇编

```bash
clang -S add.c -o add.s
```

指定目标架构：

```bash
clang -S add.c -target arm64-apple-ios17.0 -o add_arm64.s
```

这可以帮助你观察 LLVM 后端如何把 IR 降到 ARM64 指令。

---

## 6. LLVM 常用工具

### 6.1 clang

`clang` 是最常用入口。

```bash
clang main.c -o main
clang -O2 main.c -o main
clang -Wall -Wextra main.c -o main
```

### 6.2 opt

`opt` 用于运行 LLVM IR 优化和分析 Pass。

```bash
opt -S -passes=mem2reg input.ll -o output.ll
opt -S -passes='default<O2>' input.ll -o output.ll
```

常见用途：

- 单独观察某个优化 Pass 的效果
- 验证自定义 Pass
- 输出分析结果
- 调试优化管线

### 6.3 llc

`llc` 把 LLVM IR 编译成汇编或目标机器代码。

```bash
llc input.ll -o output.s
llc -march=aarch64 input.ll -o output_arm64.s
```

### 6.4 llvm-as 与 llvm-dis

文本 IR 转 Bitcode：

```bash
llvm-as input.ll -o input.bc
```

Bitcode 转文本 IR：

```bash
llvm-dis input.bc -o input.ll
```

### 6.5 lli

`lli` 是 LLVM IR 解释器和 JIT 执行工具。

```bash
lli input.ll
```

适合快速验证简单 IR。

### 6.6 llvm-objdump

查看对象文件或可执行文件反汇编：

```bash
llvm-objdump -d main
llvm-objdump --macho -d AppBinary
```

### 6.7 llvm-nm

查看符号表：

```bash
llvm-nm main
```

### 6.8 llvm-profdata 与 llvm-cov

用于覆盖率和 Profile-Guided Optimization。

```bash
llvm-profdata merge default.profraw -o default.profdata
llvm-cov show ./app -instr-profile=default.profdata
```

---

## 7. Pass 机制

### 7.1 Pass 是什么

Pass 是 LLVM 优化和分析的基本单位。一个 Pass 可以遍历 Module、Function、Loop 或 BasicBlock，然后做分析或修改。

Pass 可以分为两类：

- Analysis Pass：只分析，不修改 IR
- Transform Pass：修改 IR

常见粒度：

| Pass 类型 | 作用范围 |
| --- | --- |
| Module Pass | 整个模块 |
| Function Pass | 单个函数 |
| Loop Pass | 单个循环 |
| BasicBlock Pass | 单个基本块 |

### 7.2 Pass Manager

LLVM 使用 Pass Manager 管理 Pass 的执行顺序和分析结果缓存。

现代 LLVM 主要使用 New Pass Manager。

示例：

```bash
opt -S -passes=instcombine input.ll -o output.ll
```

多个 Pass：

```bash
opt -S -passes='mem2reg,instcombine,simplifycfg' input.ll -o output.ll
```

默认 O2 管线：

```bash
opt -S -passes='default<O2>' input.ll -o output.ll
```

### 7.3 常见 Pass

#### mem2reg

把符合条件的栈变量提升为 SSA 寄存器。

优化前：

```llvm
%x = alloca i32
store i32 1, ptr %x
%v = load i32, ptr %x
ret i32 %v
```

优化后：

```llvm
ret i32 1
```

#### instcombine

合并和简化指令。

```c
int f(int x) {
    return x + 0;
}
```

优化后相当于：

```c
int f(int x) {
    return x;
}
```

#### simplifycfg

简化控制流图。

例如把无意义的跳转块合并，把简单分支转换为 `select`。

#### inline

函数内联，把被调用函数体展开到调用点。

```c
static int add(int a, int b) {
    return a + b;
}

int test(int x) {
    return add(x, 1);
}
```

内联后可能变成：

```c
int test(int x) {
    return x + 1;
}
```

### 7.4 自定义 Pass 示例

下面是一个概念性 Function Pass，用于打印函数名。不同 LLVM 版本的 CMake 和 API 细节会变化，但核心思想一致。

```cpp
#include "llvm/IR/PassManager.h"
#include "llvm/IR/Function.h"
#include "llvm/Support/raw_ostream.h"

using namespace llvm;

namespace {

struct PrintFunctionNamePass : PassInfoMixin<PrintFunctionNamePass> {
    PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
        errs() << "Function: " << F.getName() << "\n";
        return PreservedAnalyses::all();
    }
};

}
```

如果是 Transform Pass，返回值需要准确声明哪些分析结果仍然有效：

```cpp
return PreservedAnalyses::none();
```

如果没有修改 IR：

```cpp
return PreservedAnalyses::all();
```

---

## 8. LLVM IR 示例详解

### 8.1 函数调用

C 代码：

```c
int add(int a, int b) {
    return a + b;
}

int caller() {
    return add(1, 2);
}
```

IR：

```llvm
define i32 @add(i32 %a, i32 %b) {
entry:
  %sum = add nsw i32 %a, %b
  ret i32 %sum
}

define i32 @caller() {
entry:
  %result = call i32 @add(i32 1, i32 2)
  ret i32 %result
}
```

在 `-O2` 下，`caller` 可能直接优化为：

```llvm
define i32 @caller() {
entry:
  ret i32 3
}
```

这包括函数内联和常量折叠。

### 8.2 结构体

C 代码：

```c
struct Point {
    int x;
    int y;
};

int get_x(struct Point *p) {
    return p->x;
}
```

IR 可能类似：

```llvm
%struct.Point = type { i32, i32 }

define i32 @get_x(ptr %p) {
entry:
  %x_ptr = getelementptr inbounds %struct.Point, ptr %p, i32 0, i32 0
  %x = load i32, ptr %x_ptr
  ret i32 %x
}
```

`getelementptr` 简称 GEP，用于基于类型计算地址。

### 8.3 getelementptr

`getelementptr` 不直接访问内存，只计算地址。

```llvm
%field_ptr = getelementptr inbounds %struct.Point, ptr %p, i32 0, i32 1
```

含义：

- `%p` 指向一个 `Point`
- 第一个 `0` 表示仍在当前对象内
- 第二个 `1` 表示访问结构体第 1 个字段，也就是 `y`

数组示例：

```c
int get_item(int *arr, int index) {
    return arr[index];
}
```

IR：

```llvm
define i32 @get_item(ptr %arr, i32 %index) {
entry:
  %item_ptr = getelementptr inbounds i32, ptr %arr, i32 %index
  %item = load i32, ptr %item_ptr
  ret i32 %item
}
```

### 8.4 全局变量

C 代码：

```c
int counter = 0;

void inc() {
    counter++;
}
```

IR：

```llvm
@counter = global i32 0

define void @inc() {
entry:
  %old = load i32, ptr @counter
  %new = add nsw i32 %old, 1
  store i32 %new, ptr @counter
  ret void
}
```

### 8.5 函数属性

LLVM IR 可以给函数和参数添加属性，帮助优化器做更大胆的优化。

```llvm
define i32 @square(i32 %x) #0 {
entry:
  %r = mul nsw i32 %x, %x
  ret i32 %r
}

attributes #0 = { noinline nounwind readnone }
```

常见属性：

- `noinline`：禁止内联
- `alwaysinline`：尽量强制内联
- `nounwind`：不会抛出异常
- `readnone`：不读写内存，仅依赖参数
- `readonly`：只读内存，不写内存
- `noreturn`：函数不返回

---

## 9. 从源码到机器码

### 9.1 编译流程分层

以 C 代码为例：

```text
main.c
  ↓ 预处理
main.i
  ↓ 编译前端
AST
  ↓ IR 生成
main.ll / main.bc
  ↓ 优化
optimized.ll
  ↓ 指令选择
Machine IR
  ↓ 寄存器分配
Machine Code
  ↓ 汇编器
main.o
  ↓ 链接器
main
```

### 9.2 预处理

```bash
clang -E main.c -o main.i
```

预处理负责：

- 展开宏
- 处理 `#include`
- 处理条件编译
- 删除注释

### 9.3 编译到 IR

```bash
clang -S -emit-llvm main.c -o main.ll
```

### 9.4 IR 到汇编

```bash
llc main.ll -o main.s
```

### 9.5 汇编到对象文件

```bash
clang -c main.s -o main.o
```

### 9.6 链接

```bash
clang main.o -o main
```

链接器负责：

- 合并多个对象文件
- 解析符号引用
- 处理静态库和动态库
- 生成最终可执行文件或动态库

---

## 10. 优化等级

### 10.1 常见优化等级

| 选项 | 说明 |
| --- | --- |
| `-O0` | 基本不优化，适合调试 |
| `-O1` | 轻量优化 |
| `-O2` | 常用优化等级，平衡编译时间和运行性能 |
| `-O3` | 更激进优化，可能增加代码体积 |
| `-Os` | 优化性能并尽量减小体积 |
| `-Oz` | 更强调体积 |
| `-Ofast` | 允许突破部分语言标准约束，常见于浮点优化 |

Debug 通常使用 `-O0` 或较低优化。Release 通常使用 `-O`、`-Osize` 或对应平台默认配置。

### 10.2 为什么 Debug 和 Release 差异大

Debug 编译通常保留更多结构：

- 局部变量更容易在调试器中看到
- 函数不容易被内联
- 控制流更接近源码
- 栈帧更稳定
- 编译速度更快

Release 编译会积极优化：

- 函数内联
- 删除无用变量和分支
- 循环优化
- 常量传播
- 寄存器分配更激进
- 调试时变量可能显示为 optimized out

所以 Release 下崩溃栈、断点行为和变量可见性经常与 Debug 不同。

### 10.3 优化示例

C 代码：

```c
int test() {
    int a = 1;
    int b = 2;
    int c = a + b;
    return c;
}
```

`-O0` 可能保留 `alloca/load/store`。

`-O2` 可能直接变成：

```llvm
define i32 @test() {
entry:
  ret i32 3
}
```

---

## 11. LLVM 与 Swift

### 11.1 Swift 编译流程

Swift 不会直接从源码生成 LLVM IR。它有自己的中间表示 SIL。

```text
Swift Source
  ↓ Parser
AST
  ↓ Type Checker
Typed AST
  ↓ SILGen
Raw SIL
  ↓ SIL Optimizer
Optimized SIL
  ↓ IRGen
LLVM IR
  ↓ LLVM Optimizer
Machine Code
```

SIL 比 LLVM IR 更了解 Swift 语义：

- ARC
- 泛型
- 协议见证表
- 值语义
- 闭包捕获
- async/await
- 错误处理
- 所有权模型

### 11.2 查看 Swift 的 SIL

示例：

```swift
func add(_ a: Int, _ b: Int) -> Int {
    a + b
}
```

命令：

```bash
swiftc -emit-sil add.swift
swiftc -emit-sil -O add.swift
```

查看 LLVM IR：

```bash
swiftc -emit-ir add.swift
swiftc -emit-ir -O add.swift
```

生成汇编：

```bash
swiftc -emit-assembly add.swift
```

### 11.3 Swift 泛型与优化

Swift 泛型示例：

```swift
func identity<T>(_ value: T) -> T {
    value
}

let x = identity(10)
```

编译器可能对具体类型做泛型特化，把通用泛型调用优化成具体类型版本。

概念上：

```swift
func identity_Int(_ value: Int) -> Int {
    value
}
```

这可以减少间接调用和元数据传递。

### 11.4 ARC 优化

Swift 和 Objective-C 都涉及引用计数。编译器会插入 retain/release，再通过优化删除冗余操作。

Swift 代码：

```swift
final class Box {
    let value: Int
    init(_ value: Int) {
        self.value = value
    }
}

func make() -> Int {
    let box = Box(10)
    return box.value
}
```

优化后，编译器可能通过逃逸分析发现 `box` 没有逃逸，从而减少堆分配或引用计数成本。具体结果依赖编译器版本、优化等级和上下文。

---

## 12. LLVM 与 Objective-C

### 12.1 Objective-C 编译流程

Objective-C 使用 Clang 前端。

```text
Objective-C Source
  ↓ Clang Parser
AST
  ↓ CodeGen
LLVM IR
  ↓ LLVM Backend
Machine Code
```

Objective-C 的动态特性由 Runtime 支撑，例如：

- `objc_msgSend`
- 类和元类结构
- 方法列表
- 属性和协议元数据
- Category
- Associated Object
- Method Swizzling

### 12.2 消息发送

Objective-C 代码：

```objc
[person sayHello];
```

概念上会变成：

```c
objc_msgSend(person, @selector(sayHello));
```

在 LLVM IR 中通常能看到对 Runtime 函数和选择子的引用。最终代码生成时，消息发送会变成 ABI 约定下的函数调用。

### 12.3 ARC

Objective-C ARC 由 Clang 负责插入内存管理调用。

示例：

```objc
- (NSString *)name {
    NSString *value = [[NSString alloc] initWithString:@"Tom"];
    return value;
}
```

Clang 会根据 ARC 规则插入 retain、release、autorelease 等操作，然后 LLVM 优化器和 ARC 优化 Pass 会尝试减少冗余。

---

## 13. Sanitizer 与插桩

### 13.1 Sanitizer 是什么

Sanitizer 是 LLVM 生态中非常重要的动态检测工具。它通过编译期插桩和运行时库配合，发现程序问题。

常见 Sanitizer：

- Address Sanitizer：检测越界、Use After Free 等内存问题
- Thread Sanitizer：检测数据竞争
- Undefined Behavior Sanitizer：检测未定义行为
- Memory Sanitizer：检测未初始化内存读取

### 13.2 Address Sanitizer 示例

C 代码：

```c
#include <stdlib.h>

int main() {
    int *p = malloc(sizeof(int));
    free(p);
    return *p;
}
```

编译：

```bash
clang -fsanitize=address main.c -o main
./main
```

ASan 会在编译期插入检查，在运行时发现 use-after-free。

### 13.3 Thread Sanitizer 示例

```c
#include <pthread.h>

int counter = 0;

void *work(void *) {
    counter++;
    return 0;
}

int main() {
    pthread_t t1, t2;
    pthread_create(&t1, 0, work, 0);
    pthread_create(&t2, 0, work, 0);
    pthread_join(t1, 0);
    pthread_join(t2, 0);
    return counter;
}
```

编译：

```bash
clang -fsanitize=thread main.c -o main
```

TSan 会检测多个线程未同步访问 `counter`。

### 13.4 插桩思想

插桩就是在原程序中插入额外代码。

原始代码：

```c
arr[i] = 10;
```

插桩后概念上类似：

```c
check_address(&arr[i], sizeof(int));
arr[i] = 10;
```

LLVM Pass 很适合做插桩，因为它能统一处理多种前端生成的 IR。

---

## 14. JIT 与 ORC

### 14.1 JIT 是什么

JIT 是 Just-In-Time Compilation，即运行时即时编译。

普通 AOT 编译：

```text
源码 -> 编译器 -> 可执行文件 -> 运行
```

JIT：

```text
源码 / IR -> 程序运行中编译 -> 机器码 -> 立即执行
```

LLVM 支持 JIT，现代主要使用 ORC JIT。

### 14.2 JIT 适合什么场景

- 脚本语言运行时
- 数据库查询编译
- 数值计算
- GPU 或异构计算
- 动态规则执行
- REPL

### 14.3 简化理解

JIT 的核心流程：

```text
构造 LLVM IR
  ↓
优化 IR
  ↓
为当前机器生成代码
  ↓
映射到可执行内存
  ↓
通过函数指针调用
```

---

## 15. LLVM 后端基础

### 15.1 后端要解决的问题

后端不是简单地把 IR 一条条翻译成汇编。它需要处理：

- 指令选择
- 调用约定
- 寄存器分配
- 栈帧布局
- 指令调度
- 目标机器特性
- 重定位
- 调试信息

### 15.2 指令选择

LLVM IR：

```llvm
%sum = add i32 %a, %b
```

在 ARM64 上可能变成：

```asm
add w0, w0, w1
```

在 x86_64 上可能变成：

```asm
leal (%rdi,%rsi), %eax
```

同一个 IR，在不同目标平台上可以生成不同机器指令。

### 15.3 寄存器分配

LLVM IR 中有无限虚拟寄存器：

```llvm
%a1 = add i32 %a, 1
%a2 = add i32 %a1, 2
%a3 = add i32 %a2, 3
```

真实 CPU 寄存器数量有限。后端必须把虚拟寄存器映射到物理寄存器。如果寄存器不够，就会产生 spill，把值临时存到栈上。

寄存器分配对性能影响很大。

### 15.4 ABI 与调用约定

函数调用不只是跳转。它还涉及：

- 参数放在哪些寄存器或栈位置
- 返回值放在哪里
- 哪些寄存器由调用者保存
- 哪些寄存器由被调用者保存
- 栈如何对齐
- 结构体如何传参

Apple 平台上的 ARM64 ABI 会影响 Swift、Objective-C、C 函数最终如何调用。

---

## 16. 调试信息

### 16.1 调试信息的作用

编译器生成机器码后，源码结构已经大幅变化。调试信息用于建立源码与机器码之间的映射。

调试信息帮助调试器知道：

- 某条机器指令对应源码哪一行
- 局部变量在哪里
- 函数边界在哪里
- 类型信息是什么
- 内联函数调用栈如何还原

### 16.2 生成调试信息

```bash
clang -g main.c -o main
```

结合优化：

```bash
clang -g -O2 main.c -o main
```

`-g -O2` 下可以调试，但变量可能被优化掉，代码执行顺序也可能与源码不完全一致。

### 16.3 LLVM IR 中的调试元数据

LLVM IR 中调试信息以 metadata 形式存在：

```llvm
!dbg !12
```

这种元数据会在优化和后端代码生成中尽量保留。

---

## 17. LTO 与 ThinLTO

### 17.1 传统编译的限制

传统编译以源文件为单位：

```text
a.c -> a.o
b.c -> b.o
链接 a.o b.o
```

编译 `a.c` 时，编译器通常看不到 `b.c` 内部实现，因此跨文件优化受限。

### 17.2 LTO

LTO 是 Link-Time Optimization，链接时优化。

它把更多 IR 信息保留到链接阶段，让优化器可以跨编译单元工作。

能够做的优化：

- 跨文件函数内联
- 全局死代码删除
- 跨模块常量传播
- 更精确的可见性分析

### 17.3 ThinLTO

完整 LTO 可能消耗大量时间和内存。ThinLTO 使用摘要信息实现更可扩展的跨模块优化。

ThinLTO 更适合大型工程。

---

## 18. 学习 LLVM 的路线

### 18.1 第一阶段：建立编译器基础

目标是理解基本编译流程。

需要掌握：

- C 语言基础
- 基本数据结构
- 词法分析、语法分析、语义分析概念
- AST
- 中间表示
- 控制流图
- SSA

建议练习：

1. 写一个简单表达式解释器。
2. 支持加减乘除和括号。
3. 增加变量和赋值。
4. 增加 `if` 和 `while`。
5. 把 AST 打印出来。

### 18.2 第二阶段：学习 Clang 和 LLVM 工具

目标是会观察编译器中间结果。

练习：

```bash
clang -E main.c -o main.i
clang -Xclang -ast-dump -fsyntax-only main.c
clang -S -emit-llvm main.c -o main.ll
clang -S main.c -o main.s
```

重点观察：

- 宏展开前后有什么区别
- AST 如何表达函数、变量和表达式
- `-O0` 和 `-O2` 的 IR 差异
- IR 和汇编之间如何对应

### 18.3 第三阶段：系统学习 LLVM IR

目标是看懂常见 IR。

重点：

- 类型系统
- Module、Function、BasicBlock
- `alloca/load/store`
- `call`
- `br`
- `phi`
- `getelementptr`
- `icmp/fcmp`
- `select`
- 函数属性
- metadata

建议练习：

1. 写 C 代码。
2. 生成 `-O0` IR。
3. 生成 `-O2` IR。
4. 手动解释每条 IR。
5. 用 `opt` 单独跑 Pass。

### 18.4 第四阶段：学习 Pass

目标是理解优化器如何工作。

练习顺序：

1. 用 `opt -passes=mem2reg` 观察栈变量提升。
2. 用 `opt -passes=instcombine` 观察指令合并。
3. 用 `opt -passes=simplifycfg` 观察控制流简化。
4. 写一个打印函数名的 Pass。
5. 写一个统计指令数量的 Pass。
6. 写一个简单插桩 Pass。

### 18.5 第五阶段：学习 Clang Tooling

如果你的目标是代码分析、自动重构、代码规范检查，应该重点学习 Clang Tooling。

方向：

- LibClang
- LibTooling
- AST Matcher
- clang-tidy
- clang-format
- clang static analyzer

示例任务：

- 找出所有 Objective-C 方法声明
- 找出某个类的所有调用点
- 检查禁止使用的 API
- 自动把旧 API 替换成新 API
- 为函数自动添加日志

### 18.6 第六阶段：深入后端

如果你的目标是编译器后端或目标架构支持，需要学习：

- TableGen
- SelectionDAG
- GlobalISel
- Machine IR
- Register Allocation
- Calling Convention
- TargetLowering
- MC Layer

这是 LLVM 中难度较高的部分，建议在熟悉 IR 和 Pass 后再进入。

---

## 19. 实战示例：观察优化

### 19.1 准备代码

创建 `example.c`：

```c
static int add(int a, int b) {
    return a + b;
}

int test() {
    int x = 1;
    int y = 2;
    return add(x, y);
}
```

### 19.2 生成 O0 IR

```bash
clang -S -emit-llvm -O0 example.c -o example_O0.ll
```

你可能看到：

- `alloca`
- `store`
- `load`
- 对 `add` 的 `call`

### 19.3 生成 O2 IR

```bash
clang -S -emit-llvm -O2 example.c -o example_O2.ll
```

你可能看到 `test` 直接返回常量：

```llvm
define i32 @test() {
entry:
  ret i32 3
}
```

### 19.4 分析发生了什么

优化过程可以理解为：

```text
局部变量 x/y
  ↓ mem2reg
SSA 值
  ↓ inline
add 函数被内联
  ↓ constant propagation
1 和 2 传播到加法
  ↓ constant folding
1 + 2 折叠为 3
  ↓ dead code elimination
删除无用函数和中间值
```

---

## 20. 实战示例：手写 LLVM IR

### 20.1 hello.ll

```llvm
@.str = private unnamed_addr constant [14 x i8] c"Hello LLVM!\0A\00"

declare i32 @printf(ptr, ...)

define i32 @main() {
entry:
  %str = getelementptr inbounds [14 x i8], ptr @.str, i32 0, i32 0
  call i32 (ptr, ...) @printf(ptr %str)
  ret i32 0
}
```

运行：

```bash
lli hello.ll
```

或编译：

```bash
clang hello.ll -o hello
./hello
```

### 20.2 手写加法函数

```llvm
define i32 @add(i32 %a, i32 %b) {
entry:
  %sum = add nsw i32 %a, %b
  ret i32 %sum
}

define i32 @main() {
entry:
  %result = call i32 @add(i32 20, i32 22)
  ret i32 %result
}
```

如果在 shell 中运行程序，进程退出码可能是 `42`。

---

## 21. 实战示例：简单插桩思想

假设我们想在每个函数入口打印函数名。

源代码：

```c
int add(int a, int b) {
    return a + b;
}

int main() {
    return add(1, 2);
}
```

插桩后的概念代码：

```c
int add(int a, int b) {
    printf("enter add\n");
    return a + b;
}

int main() {
    printf("enter main\n");
    return add(1, 2);
}
```

LLVM Pass 可以做的事情：

1. 遍历 Module 中的每个 Function。
2. 跳过声明函数，例如 `printf`。
3. 找到函数入口 BasicBlock。
4. 在第一条有效指令前插入 `printf` 调用。
5. 创建或复用全局字符串常量。

伪代码：

```cpp
for (Function &F : M) {
    if (F.isDeclaration()) {
        continue;
    }

    BasicBlock &Entry = F.getEntryBlock();
    IRBuilder<> Builder(&*Entry.getFirstInsertionPt());
    Builder.CreateCall(Printf, Message);
}
```

这种能力可以用于：

- 性能耗时统计
- 覆盖率统计
- 日志追踪
- 安全检查
- API 调用审计

---

## 22. Clang AST Matcher 示例

如果目标是分析源码而不是优化 IR，Clang AST Matcher 更合适。

例如查找所有函数声明：

```cpp
functionDecl().bind("func")
```

查找名为 `viewDidLoad` 的 Objective-C 方法：

```cpp
objcMethodDecl(hasName("viewDidLoad")).bind("method")
```

查找调用某个函数的表达式：

```cpp
callExpr(callee(functionDecl(hasName("dangerous_api")))).bind("call")
```

AST 层保留了源码语义，比 LLVM IR 更适合：

- 代码规范检查
- API 迁移
- 自动重构
- 源码级诊断

LLVM IR 更适合：

- 性能优化
- 插桩
- 数据流分析
- 后端代码生成
- 跨语言统一分析

---

## 23. LLVM 中的重要概念

### 23.1 Undefined Behavior

C/C++ 中的未定义行为会给优化器很大空间。

示例：

```c
int f(int x) {
    return x + 1 > x;
}
```

如果有符号整数溢出是未定义行为，编译器可能假设 `x + 1` 不会溢出，从而把函数优化为总是返回 `1`。

这类优化在安全、逆向和 Bug 分析中非常重要。

### 23.2 Alias Analysis

Alias Analysis 用于判断两个指针是否可能指向同一块内存。

```c
void f(int *a, int *b) {
    *a = 1;
    *b = 2;
}
```

如果 `a` 和 `b` 可能别名，优化器必须保守。如果能证明它们不别名，就能做更多重排和优化。

### 23.3 Escape Analysis

逃逸分析判断对象或指针是否逃出当前作用域。

如果对象没有逃逸，编译器可能：

- 消除堆分配
- 把对象拆成标量
- 删除不必要的引用计数
- 减少同步开销

### 23.4 Data Flow Analysis

数据流分析研究值如何在程序中传播。

常见问题：

- 某个变量在这里是否一定被赋值？
- 某个表达式是否总是同一个值？
- 某个内存写入后是否还有读取？
- 某个分支是否永远不会执行？

### 23.5 Control Flow Graph

控制流图把基本块作为节点，把跳转关系作为边。

```text
entry
  ↓
condition
  ├── then
  └── else
       ↓
      merge
```

很多优化依赖 CFG：

- 删除不可达块
- 合并基本块
- 循环识别
- 支配树计算
- 分支概率分析

### 23.6 Dominator Tree

如果所有从入口到 B 的路径都必须经过 A，则 A 支配 B。

支配关系用于：

- SSA 构造
- mem2reg
- 循环优化
- 代码移动
- 死代码删除

---

## 24. 常见学习误区

### 24.1 一开始就读 LLVM 源码

LLVM 源码巨大，直接阅读容易迷失。更好的顺序是：

1. 先会用工具。
2. 再读 IR。
3. 再观察 Pass。
4. 最后按问题读源码。

### 24.2 把 LLVM IR 当成汇编

LLVM IR 接近底层，但不是汇编。它有无限虚拟寄存器、强类型、SSA、目标无关抽象。

### 24.3 忽略优化等级

很多现象只有在特定优化等级下出现。分析问题时要明确：

- 编译器版本
- 目标平台
- 优化等级
- Debug 信息选项
- 是否启用 LTO

### 24.4 用 IR 做源码级重构

如果你要保留注释、格式、宏和源码结构，应该使用 Clang Tooling，而不是 LLVM IR。

IR 层已经丢失大量源码信息，适合优化和低层分析。

### 24.5 忽略 ABI

看汇编时不能只看指令，还要理解 ABI。参数、返回值、栈布局、寄存器保存规则都由 ABI 决定。

---

## 25. 推荐实践项目

### 25.1 入门项目

- 写一个四则运算语言解释器
- 用 LLVM IR 生成表达式计算代码
- 手写 `.ll` 文件并用 `lli` 执行
- 对比 `-O0` 和 `-O2` IR
- 用 `opt` 跑单个 Pass

### 25.2 中级项目

- 写一个 LLVM Pass 打印函数名
- 写一个 Pass 统计指令数量
- 写一个 Pass 在函数入口插入日志
- 写一个 Clang Tool 检查禁止 API
- 写一个 clang-tidy 规则

### 25.3 进阶项目

- 实现一个玩具语言前端，生成 LLVM IR
- 支持变量、函数、if、while
- 支持简单类型检查
- 使用 ORC JIT 执行代码
- 实现简单优化 Pass
- 为某个小型目标架构写后端实验

### 25.4 iOS 方向项目

- 分析 Objective-C 方法调用生成的 IR 和汇编
- 对比 Swift `final`、`private`、协议调用的 SIL 和 IR
- 观察 ARC retain/release 优化
- 使用 Sanitizer 复现并定位内存错误
- 用 Clang Tooling 检查项目中的危险 API
- 分析 Release 崩溃栈和优化的关系

---

## 26. 常见命令速查

```bash
# 预处理
clang -E main.c -o main.i

# AST
clang -Xclang -ast-dump -fsyntax-only main.c

# 生成 LLVM IR
clang -S -emit-llvm main.c -o main.ll

# 生成优化后 LLVM IR
clang -S -emit-llvm -O2 main.c -o main_O2.ll

# 生成 Bitcode
clang -c -emit-llvm main.c -o main.bc

# Bitcode 转文本 IR
llvm-dis main.bc -o main.ll

# 文本 IR 转 Bitcode
llvm-as main.ll -o main.bc

# 运行优化 Pass
opt -S -passes=mem2reg main.ll -o main_mem2reg.ll

# IR 生成汇编
llc main.ll -o main.s

# 反汇编
llvm-objdump -d main

# 查看符号
llvm-nm main

# Swift SIL
swiftc -emit-sil main.swift

# Swift LLVM IR
swiftc -emit-ir main.swift

# Swift 汇编
swiftc -emit-assembly main.swift
```

---

## 27. 建议的学习顺序

如果你是 iOS 或客户端开发者，可以按这个顺序学习：

1. 复习 C 语言、指针、结构体和函数调用。
2. 学会用 `clang` 生成 AST、LLVM IR 和汇编。
3. 阅读简单 C 函数的 `-O0` IR。
4. 阅读同一函数的 `-O2` IR。
5. 学习 SSA、BasicBlock、PHI、GEP。
6. 用 `opt` 跑 `mem2reg`、`instcombine`、`simplifycfg`。
7. 学习 Swift SIL，再看 Swift 生成的 LLVM IR。
8. 学习 Objective-C 消息发送和 ARC 在 IR 中的大致形态。
9. 使用 Sanitizer 理解编译期插桩。
10. 根据目标选择 Clang Tooling、Pass、JIT 或后端方向深入。

如果目标是写编译器：

1. 写解释器。
2. 写 AST。
3. 做类型检查。
4. 生成 LLVM IR。
5. 接入 JIT。
6. 增加函数、控制流和作用域。
7. 增加简单优化。
8. 学习运行时和 GC 或引用计数。

如果目标是做静态分析：

1. 学习 Clang AST。
2. 学习 AST Matcher。
3. 学习 LibTooling。
4. 写简单检查器。
5. 学习 clang-tidy。
6. 根据需要进入 LLVM IR 数据流分析。

---

## 28. 最小学习闭环

学习 LLVM 最重要的是形成闭环：

```text
写一段小代码
  ↓
生成 IR
  ↓
猜测优化结果
  ↓
运行 opt 或 clang -O2
  ↓
对比差异
  ↓
解释原因
```

示例：

```c
int f(int x) {
    int y = x * 2;
    return y / 2;
}
```

你可以思考：

- 编译器能不能把它优化成 `return x`？
- 如果 `x * 2` 可能溢出怎么办？
- 使用有符号整数和无符号整数结果是否不同？
- `-O0` 和 `-O2` 有什么差别？

这种问题会迫使你理解语言语义、未定义行为、IR 标记和优化器假设。

---

## 29. 总结

LLVM 是现代编译器工程中最重要的基础设施之一。它的核心不是某一个工具，而是一套围绕 LLVM IR 组织起来的模块化体系。

学习 LLVM 应该抓住几条主线：

- 前端负责语言语义
- LLVM IR 负责统一表达程序
- Pass 负责分析和优化
- 后端负责生成目标机器代码
- 工具链负责把这些能力组合到真实工程中

对 iOS 开发者来说，LLVM 不是遥远的底层技术。Clang、Swift、LLDB、Sanitizer、Release 优化、崩溃分析、静态检查和性能优化都与它有关。

最有效的学习方式不是只读文档，而是持续写小代码、生成 IR、对比优化、阅读汇编，并把观察结果和语言语义联系起来。只要能稳定看懂简单 IR，再逐步理解 Pass、SIL、Clang AST 和后端，LLVM 的知识体系就会越来越清晰。
