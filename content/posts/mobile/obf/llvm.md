---
title: 初探LLVM框架
date: 2025-11-30 23:30:38
categories: Mobile
tags: 
  - 加固开发
mathjax:
top:
---

## 简介

LLVM（Low Level Virtual Machine）是一个开源的编译器基础设施项目，旨在提供可重用的编译器和工具链技术，支持编译器的开发、优化和分析，从而帮助开发者创建高效的程序和工具。

目前许多现代编程语言（例如 Rust、Swift、Julia 等）都使用 LLVM 作为其核心的优化和代码生成后端。

本文所讨论的 LLVM 版本为 LLVM-18.1.8，与旧版本相比，引入了包括新的 Pass 管理器在内的一系列更新与改进。

LLVM 三段式架构：

- **前端**：解析源代码，由语法分析器和语义分析协同工作，检查错误，并构建特定于语言的抽象语法树来表示输入代码。然后将分析好的代码转化为 LLVM 的中间表示 IR。
- **优化器**：通过 Pass 对中间代码 IR 进行优化，改善改名的运行时间使代码更高效。
- **后端**：负责将优化器优化后的中间代码 IR 转换为目标机器的代码。

![](assets/LLVM的结构.png)


LLVM IR 作为整个架构的核心枢纽，具有以下关键特性：

- 强类型静态单赋值（SSA）形式
- 三种等价表达方式：内存架构、Bitcode二进制格式、可读文本格式
- 丰富的元数据支持调试和优化信息

LLVM 和传统的 gcc 的区别：

- 架构和设计
  - LLVM编译器是基于模块化、可扩展的设计，它将编译过程划分为多个独立的阶段，。并使用中间表示（IR）作为通用的数据结构进行代码优化和生成。而 GCC 编译器则是集成了多个前端和后端的传统编译器，其设计更加紧密一体化
- 开发语言和前端支持
  - LLVM 编译器使用 C++ 开发，并提供了广泛的前端支持，可以处理多种编程语言（如C、C++、Rust等），这使得开发者能够使用统一的编译框架来处理不同的语言。GCC编译器则使用C语言开发，并对各种编程语言提供了广泛的前端支持。
- 优化功能
  - LLVM 编译器一起高度模块化的中间表示（IR）为基础，具备强大的代码优化能力。同时，LLVM的设计也使得优化部分可以在编译过程的各个阶段进行，从而实现更细粒度的优化。GCC 编译器也提供了一系列的优化选项，但其优化能力相对较低。


## 环境搭建

### 预编译包安装

在系统上直接通过包管理安装，这样会直接安装系统默认版本的 LLVM。

```shell
apt install clang
apt install llvm
```

### 编译安装

下载项目，然后在构建目录中运行以下命令，根据 cmake 编译安装指定架构的 Release 版本的 llvm 和 clang。

```shell
mkdir build && cd build
cmake -G "Unix Makefiles" \
    -DLLVM_ENABLE_PROJECTS="clang;llvm;" \
    -DCMAKE_BUILD_TYPE=Release \
    -DLLVM_TARGETS_TO_BUILD="X86" \
    -DBUILD_SHARED_LIBS=On \
    -DLLVM_ENABLE_LLD=ON \
    ../llvm
make -j$(nproc)
make install
```

## 基础知识

LLVM 编译流程可以大致分为前端处理、IR 优化和后端生成三个阶段，下面是具体步骤。

### 前端的语法分析

将源代码转经过 Clang 前端的预处理、语法分析、语义分析转换成 AST 树。

```shell
clang -Xclang -ast-dump -fsyntax-only test.c
```

### 前端生成中间代码

根据内存中的抽象语法树 AST 生成 LLVM IR 中间代码（有的比较新的编译器还会先将 AST 转换为 MLIR 再转换为 IR）。

```shell
clang -S -emit-llvm test.c
```

### IR优化

通过 opt 优化 LLVM IR。

```shell
opt test.ll -S --O3
clang -S -emit-llvm -O3 test.c
```

### 运行IR

```
lli test.ll
```

### 后端生成汇编代码

 将 LLVM IR 转成目标机器汇编代码：

```shell
llc test.ll
```

### bc 生成 ll

```shell
llvm-dis demo.bc
```

### ll 生成 bc

```shell
llvm-as demo.ll -o demo.bc
```


## LLVM IR

IR 是一种低级编程语言。能够代表高层次的想法。即高级语言可以清晰地映射到 IR。实现高效的代码优化。

![](assets/LLVM%20Flow.drawio.png)


### 基本概念

高级语言经过 Clang 前端会将代码解析为平台无关的中间表示语言（IR），使编译器能够在编译、链接、以及代码生成的各个阶段忽略语言特性，进行全面有效的优化和分析。

LLVM 基于统一的中间表示来实现优化遍，中间表示采用静态单赋值形式，该形式的虚拟指令集能够高效的表示高级语言，具有灵活性好、类型安全、底层操作等特点。如图所示，当同一变量出现多次赋值时，通过 SSA 变量重命名的方式加以区分，可以避免出现多次定义的情况。

IR 的设计很大程度体现着 LLVM 插件化、模块化的设计哲学，LLVM 的各种 Pass 其实都是作用在 LLVM IR 上的。通常情况下，设计一门新的编程语言只需要完成能够生成 LLVM IR 的编译器前端即可，然后就可以轻松使用 LLVM IR 的各种编译优化、JIT 支持、目标代码生成等功能。

LLVM 目标无关代码生成器是一个框架，它提供了一套可重用组件，用于将 LLVM 内部表示转换为指定目标的机器代码，可以是汇编形式（适用于静态编译器），也可以是二进制机器代码格式（适用于 JIT 编译器）。

LLVM 后端的主要功能是代码生成，因此也叫代码生成器。后端包括若干个代码生成分析转换 Pass 将 LLVM IR 转换成特定目标架构的机器代码。

- 源码结构：
  - 特定目标的抽象目标描述接口的实现。这些机器描述使用 LLVM 提供的组件，并可以选择提供定制的特定于目标的传递，为特定目标构建完整的代码生成器。目标描述位于`lib/Target`中。
  - 用于实现本机代码生成的各个阶段（寄存器分配、调度、堆栈帧表示等）的目标无关算法。此代码位于`lib/COdeGen`中。
  - 目标独立组件 JIT 组件。LLVM JIT 完全独立于目标（使用 TargetJITInfo 结构来处理特定于目标的问题。独立于目标的 JIT 的代码位于`lib/ExecvtionEngine/JIT`中）

LLVM IR 里面的变量不等于编译代码中的变量，源代码中的局部变量存放在栈内存中，需要用 LLVM 的栈分配指令`alloca`分配栈内存，LLVM IR 变量存放分配后的 位置信息，类似于原变量的指针。对于原始变量的存取则需要内存操作指令`load`和`store`指令。

![](assets/LLVM%20IR.drawio.png)

Module 代表一个变异模块（如一个 cpp 文件），Function 就是一个函数，BasicBlock 指的是可连续执行的基本块（没有跳转等中断的指令集合），在 Pass 中可以选一个维度进行操作，比如可以遍历 Function 对每个 Function 进行操作。

### IR表示

LLVM IR 有三种形式：

- 内存中的表示形式
- bitcode 形式
- LLVM 汇编文件格式

IR表示：

- Module类，Module 可以理解为一个完整的编译单元。一般来说，这个编译单元就是一个源码文件。
- Function类，这个类顾名思义就是对应于一个函数单元。Function 可以描述两种情况，分别是函数定义和函数声明。
- BasicBlock类，这个类表示一个基本代码块，“基本代码块”就是一段没有控制流逻辑的基本流程，就相当于程序流程图中的基本过程。
- Instruction类，指令类就是 LLVM 中定义的基本操作，比如加减乘除这种算数指令、函数调用指令、跳转指令、返回指令等等。

#### 基本程序

基本概念：

- 注释：首先第一行是一个注释，IR 中注释以`;`开头，IR 只有行注释没有块注释
- 主程序：主程序是可执行程序执行的入口点，所以任何可执行程序都需要`main`函数才能运行。

```c
; main.ll
define i32 @main(){
	ret i32 0
}
```

#### 目标平台


LLVM 解决的一大问题是可移植性，通过

CPU-vendor-OS 这三者决定了一个平台，只要这三者一致，我们生成的二进制程序往往就可以确定了。

```
target triple = "x86_64-unknown-linux-gnu"
```

- 目标数据布局

除了目标平台外，我们还可以声明目标数据布局。

LLVM 也支持我们手动定制这样的数据布局：

```c
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"
```

这一长串文字就定义了目标的数据布局：

- `e`：小端序；
- `m:e`：符号表中使用 ELF 格式的符号修饰；
- `p270:32:32-p271:32:32-p272:64:64`：地址空间布局；
- i64:64：将i64类型的变量采用64位的 ABI 对齐
- f80:128：将 long double 类型的变量采用 128 位的 ABI 对齐
- n8:16:32:64：目标 CPU 的原生类型包含8位、16位、32位和64位
- `S128`：栈以 128 位自然对齐；
- 目标三元组`x86_64-unknown-linux-gnu`：
  - 架构：`x86_64`
  - 厂商：`unknown`
  - 系统：`linux`
  - ABI：`gnu`

#### 递归阶乘

```c
#include<stdio.h>
#include<stdlib.h>

int factorial(int val){
	if(val==0){
		return 1;
	}
	return val*factorial(val-1);
}

int main(){
  factorial(3);

  return 0;
}
```

```
; ModuleID = 'jc.c'
source_filename = "jc.c"
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-i128:128-f80:128-n8:16:32:64-S128"
target triple = "x86_64-unknown-linux-gnu"

; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @factorial(i32 noundef %0) #0 {
  %2 = alloca i32, align 4
  %3 = alloca i32, align 4
  store i32 %0, ptr %3, align 4
  %4 = load i32, ptr %3, align 4
  %5 = icmp eq i32 %4, 0
  br i1 %5, label %6, label %7

6:                                                ; preds = %1
  store i32 1, ptr %2, align 4
  br label %13

7:                                                ; preds = %1
  %8 = load i32, ptr %3, align 4
  %9 = load i32, ptr %3, align 4
  %10 = sub nsw i32 %9, 1
  %11 = call i32 @factorial(i32 noundef %10)
  %12 = mul nsw i32 %8, %11
  store i32 %12, ptr %2, align 4
  br label %13

13:                                               ; preds = %7, %6
  %14 = load i32, ptr %2, align 4
  ret i32 %14
}

; Function Attrs: noinline nounwind optnone uwtable
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  store i32 0, ptr %1, align 4
  %2 = call i32 @factorial(i32 noundef 3)
  ret i32 0
}

attributes #0 = { noinline nounwind optnone uwtable "frame-pointer"="all" "min-legal-vector-width"="0" "no-trapping-math"="true" "stack-protector-buffer-size"="8" "target-cpu"="x86-64" "target-features"="+cmov,+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" }

!llvm.module.flags = !{!0, !1, !2, !3, !4}
!llvm.ident = !{!5}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{i32 8, !"PIC Level", i32 2}
!2 = !{i32 7, !"PIE Level", i32 2}
!3 = !{i32 7, !"uwtable", i32 2}
!4 = !{i32 7, !"frame-pointer", i32 2}
!5 = !{!"clang version 18.1.8"}

```


### 数据表示

- 汇编层次的数据表示

IR 是最接近汇编语言的一层抽象，所以我们需要

在内存中的数据按其表示来说，一共分为两类

栈上的数据和堆上的数据

除了内存之外，还有一个存储数据的地方，那就是寄存器。因此，我们在程序中可以用来表示的数据，一共分为三类：

寄存器中的数据、栈上的数据、数据区里的数据

#### 数据区和符号表

LLVM IR 中定义一共存储在数据区中的全局变量，其格式为：

```
@global_variable = global i32 0
```

这个语句定义了一共 i32 类型的全局变量`@global_variable`，并且将其初始化为 0。

如果是只读的全局变量，也就是常量，我们可以用`constant`来代替`global`：

```
@global_constant = constant i32 0
```

这个语句定义了一共 i32 类型的全局变量`@global_constant`，并将其初始化为 0。

- 符号与符号表

讲到了数据区，就顺便讲讲符号表。在 LLVM IR 中，所有的全局变量的名称都需要用`@`开头。我们有一共这样的 LLVM IR：

```
; global_variable_test.ll
@global_variable = global i32 0

define i32 @main(){
	ret i32 0
}
```

也就是说，在之前最基本的程序的基础上，新增了一共全局变量`@global_variable`。我们将其直接编译成可执行文件：

```
clang global_var.ll -o global_variable_test
```

直接定义的全局变量，其名称会出现在符号表之中。

对于动态链接的程序而言，在程序加载、运行时，也会由动态链接器将相应的库动态链接进程序之中。

编译器生成的结果需要给链接器和动态链接器进行处理。在这一过程中，就需要符号表出马了。在上述的过程中，编译器的输入是一共编译单元，而输出是一共目标文件。如果我们在源代码中调用了别的文件中实现的函数，编译器并不知道别的函数在哪。因此，编译器选择的策略是将这个函数的调用用一共符号代替，在将来链接以及动态链接的适合，再进行替换。

粗略来讲，整体的符号处理的过程为：

1. 编译器对源代码按文件进行编译。对于每个文件中的未知函数，记录其符号；对于这个文件中实现的函数，暴露其符号
2. 链接器收集所有的目标文件，对于每个文件而言，将其记录下的未知函数的符号，与其他文件中暴露出的符号相对比，如果找到匹配的，就成功地解析了符号
3. 部分符号在程序加载、执行时，由动态链接库给出。动态链接器将在这些阶段，进行类似的符号解析。

一个符号本身，就是一个字符串。那么我们在写一个 C 语言的项目时，如果希望有的函数在别的文件中被调用，按照上述过程，似乎就是暴露一下符号就行。但是，一个 C 语言的项目，往往会链接 很多第三方库。如果我们想暴露的函数名与其他第三方库里的函数名重复了，会怎样呢？如果不加处理，链接器会直接报错。那难道我们起一个名字，需要注意与别的所有的库里的函数都不重复吗？此外，一个程序会有成千上万个符号，一些简单的，只在一个文件里用到的符号，比如说 cmp 、max，难道也要放在符号表中吗？ 在LLVM IR中，解决这些问题，与两个概念密切相关：链接与可见性，LLVM IR也提供了Linkage Type和Visibility Styles这两个修饰符来控制相应的行为。

- 链接类型

对于链接类型，我们常用的主要有声明都不加（默认为`external`）、`private`和`internal`。

什么都不加的话，就像我们刚刚那样，直接把全局变量的名字放在了符号表中。这样的话，这个函数可以在链接时被其他编译单元看到。

用`private`，则代表这个变量的名字不好出现在符号表中。我们将原来的代码改成：

```
@global_variable = private global i32 0
```

那么，用`nm`查看其编译出的可执行文件，这个变量的名字就消失了。

用`internal`则表示这个变量是以局部符号的身份出现（全局变量的局部符号，可以理解成 C 中的`static`关键字）我们将原来的代码改成：

```
@global_variable = internal global i32 0
```

那么再次将其编译成可执行程序，并用`nm`查看，可以看到这个符号。但是，在链接过程中，这个符号并不会参与符号解析。

- 可见性

可见性在实际使用中则比较少，主要分为三种`default`、`hidden`和`protected`，这里主要的区别在于符号能否被重载。`default`的符号可以被重载，而`protected`的符号则不可用；此外，`hidden`则不将变量放在动态符号表中，因此其它的模块不可以直接引用这个符号。


- 可抢占性

在我们日常看到的 LLVM IR 中，会经常见到`dso_local`这样的修饰符，在 LLVM 中被称作运行时抢 占性修饰符。如果需要深入理解这个概念，可以参考国人大神写的ELF interposition and Bsymbolic 、`-fno-semantic-interposition` 。简单来说，`dso_local`保证了程序像我们想象中的一样运行。

C语言中的static，也就是当前文件中定义，别的文件不可以用的，都会加上internal修饰符。C语言中的extern，也就是别的文件中定义的，全局变量会加上external修饰符，函数会 使用declare。C语言中定义的，可以给别的文件使用的全局变量或函数，不会加上链接类型修饰符，并且 会加上dso_local保证不会被抢占。

#### 寄存器和栈

大多数对数据的操作，如加减乘除、比大小等，都需要操作的是寄存器内的数据。之所以要不数据放在栈上，主要是因为寄存器数量不够和需要操作内存地址。

如果我们一个函数内有三四十个局部变量，但是一般CPU也就十几个寄存器，所以我们不可能把所有变量都放在寄存器中。因此我们需要把一部分数据放在内存中，栈就是一个很好的存储数据的地方；此外，有时候我们需要直接操作内存地址，但是寄存器并没有通用的地址表示，所以只能把数据放在栈上来完成对地址的操作。

因此，在不操作内存地址的前提下，栈只是寄存器的一共替代品。

如果寄存器的数量足够，并且代码中没有需要操作内存地址的时候，寄存器是足够胜任的，并且更加高效。

- 寄存器

正因为如此，LLVM IR 引入了虚拟寄存器的概念。在 LLVM IR 中，一个函数的局部变量可以是寄存器或者栈上的变量。对于寄存器而言，我们只需要普通的赋值语句一样操作，但需要注意名字必须以`%`开头：

```
%local_variable = add i32 1,2
```

此时，%local_variable这个变量就代表一个寄存器，它此时的值就是1和2相加的结果。我们 可以写一个简单的程序验证这一点：

```
;register_test.ll
define i32 @main(){
	%local_variable = add i32 1,2
	ret i32 %local_variable
}
```

我们查看其编译出的汇编代码，其主函数为：

```
main: 
	movl $2, %eax addl 
	$1, %eax retq
```

确实这个局部变量变成了寄存器`eax`。

关于寄存器，我们还需了解一点。在不同的 ABI 下，会有一些callee-saved register和caller-saved register。简单来说，就是在函数内部，某些寄存器的值不能改变。或者说，在函数返回时，某些寄存器的值要和进入函数前相同。比如，在 System V的ABI下，rbp, rbx, r12, r13, r14, r15 都需要满足这一条件，这在System V的ABI下被称作callee-saved registe。由于LLVM IR是面向多 平台的，所以我们需要一份代码适用于多种ABI。因此，LLVM IR内部自动帮我们做了这些事。如 果我们把所有没有被保留的寄存器都用光了，那么LLVM IR会帮我们把这些被保留的寄存器放在栈 上，然后继续使用这些被保留寄存器。当函数退出时，会帮我们自动从栈上获取到相应的值放回寄 存器内。

那么，如果所有通用寄存器都用光了，该怎么办？LLVM IR会帮我们把剩余的值放在栈上，但是对 我们用户而言，实际上都是虚拟寄存器，用户是感觉不到差别的。

因此，我们可以粗略地理解 LLVM IR 对寄存器的使用：当所需寄存器数量较少时，直接使用caller-saved register，即不需要保留的寄存器。当caller-saved register不够时，将caller-saved register原本的值压栈，然后使用caller-saved register。当寄存器用光以后，就把多的虚拟寄存器的值压栈。

因此，我们还可以注意到，如果在调用别的函数的过程中，如果调用方的非callee-saved register 中存有一些后续需要用到的数据，需要将这些数据放入栈上，在函数调用结束后，再从栈上将这些值放回相应的寄存器中。

我们可以写一个简单的程序验证。对于x86_64架构下，我们只需要使用15个虚拟寄存器就可以验 证这件事。

- 栈

我们之前说过，当不需要操作地址并且寄存器数量足够时，我们可以直接使用寄存器。而 LLVM IR 的策略保证了我们可以使用无数的虚拟寄存器。那么，在需要操作地址以及需要可变变量（之后会 提到为什么）时，我们就需要使用栈。

LLVM IR 对栈的使用十分简单，直接使用 alloca 指令即可。如：

```
%local_variable = alloca i32
```

就可以声明一个在栈上的变量了。

#### 数据的使用

- 全局变量和栈上变量皆制作

下面，我们就需要讲怎样使用全局变量和栈上的变量。这两种变量实际上是类似的，LLVM IR 把它们都看作指针。也就是说，对于全局变量：

```
@global_variable=global i32 0
```

和栈上变量

```
%local_variable=alloca i32
```

这两个变量实际上都是 ptr 指针，指向它们所处的一个`i32`大小的内存区域。所以，我们不能这样：

```
%1 = add i32 1,@global_variable ; Wrong!
```

因为`@global_variable`只是一个指针。

如果要操作这些值，必须使用`load`和`store`这两个命令。如果我们需要获取`@global_variable`的值，就需要

```
%1 = load i32 @global_variable
```

这个指令的意思是，把一个`ptr`指针`@global_variable`的`i32`类型的值赋给虚拟寄存器`%1`，然后我们就能愉快地

```
%2=add i32 1,%1
```

类似地，如果我们要将值存储到全局变量或栈上变量里，会需要store命令：

```
store i32 1,ptr @global_variable
```

这个代表`i32`类型的值 1 赋给`ptr`类型的全局变量`@global_variable`所指的内存区域中。

- SSA

LLVM IR是一个严格遵守SSA(Static Single Assignment)策略的语言。SSA的要求很简单：每个变量 只被赋值一次。也就是说，你不能

```
%1 = add i32 1, 2 
%1 = add i32 3, 4
```

对%1同时赋值两次是不被允许的。 SSA作为一个历史悠久的概念，已经有了相当成熟的相关技术。通过使用SSA，编译器可以进行更 好的优化，应用更成熟的算法，得到更好的结果。这里因为个人能力有限，就不再多对SSA进行介 绍。我们只需要知道，通过约束每个变量只被赋值一次，可以让LLVM更好地优化。 

上面这个例子好做，直接把3加4的结果赋值给一个新的虚拟寄存器就好了。但是，并非所有的情 况都这么简单。在一些复杂情况下，将值存储在栈上再取出来，或者使用phi指令（见之后控制 语句一章），也是一个更好的选择

### 类型系统

我们知道，汇编语言是弱类型的，我们操作汇编语言的时候，实际上考虑的是一些二进制序列。但 是，LLVM IR却是强类型的，在LLVM IR中所有变量都必须有类型。这是因为，我们在使用高级语 言编程的时候，往往都会使用强类型的语言，弱类型的语言无必要性，也不利于维护。因此，使用 强类型语言，LLVM IR可以更好地进行优化

#### 基本的数据类型

LLVM IR 中比较基本的数据类型包括：

- 空类型（void）
- 整型（iN）
- 浮点型（float、double等）

空类型一般是作为不返回值的函数的返回类型，没有特别的含义，就代表什么都没有。

整型是指i1, i8, i16, i32 , i64 这类的数据类型。这里iN的N可以是任意正整数，可以是 i3 ，i1942652 。但最常用，最符合常理的就是i1以及8的整数倍。i1有两个值：true和 false 。也就是说，下面的代码可以正确编译：

```
%boolean_variable = alloca i1 
store i1 true, ptr %boolean_variable
```

对于大于1位的整型，也就是如i8, i16等类型，我们可以直接用数字字面量赋值：

```
%integer_variable = alloca i32 
store i32 128, ptr %integer_variable 
store i32 -128, ptr %integer_variable
```

- 符号

有一点需要注意的是，在LLVM IR中，整型默认是有符号整型，也就是说我们可以直接将-128以 补码形式赋值给i32类型的变量。在LLVM IR中，整型的有无符号是体现在操作指令而非类型上 的，比方说，对于两个整型变量的除法，LLVM IR分别提供了udiv和sdiv指令分别适用于无符号 整型除法和有符号整型除法：

```
%1 = udiv i8 -6, 2 ; Get (256 - 6) / 2 = 125 
%2 = sdiv i8 -6, 2 ; Get (-6) / 2 = -3
```

我们可以用这样一个简单的程序验证：

```
; div_test.ll
define i8 @main(){
	%1 = udiv i8 -6,2
	%2 = sdiv i8 -6,2
}
```

分别将`ret`语句的参数换成`%1`和`%2`以后，将代码编译成可执行文件，在终端下运行并查看返回 值即可。 

总结一下就是，LLVM IR中的整型默认按有符号补码存储，但一个变量究竟是否要被看作有无符号 数需要看其参与的指令。

- 转换指令

与整型密切相关的就是转换指令，比如说，将`i8`类型的数`-127`转换成`i32`类型的数，将`i32`类型的数`257`转换成`i8`类型的数等。总的来说，LLVM IR中提供三种指令：`trunc .. to`指令， `zext .. to`指令和`sext .. to`指令。 将长的整型转换成短的整型很简单，直接把多余的高位去掉就行，LLVM IR提供的是trunc .. to 指令：

```
%trunc_integer = trunc i32 257 to i8 ; Trunc 32 bit 100000001 to 8 bit, get 1
```

将短的整型变成长的整型则相对比较复杂。这是因为，在补码中最高位是符号位，并不表示实际的数值。因此，如果单纯地在更高位补0，那么i8类型的-1（补码为11111111）就会变成i32的 255 。这虽然符合道理，但有时候我们需要i8类型的-1扩展到i32时仍然是-1。LLVM IR为我 们提供了两种指令：零扩展的`zext .. to`指令和符号扩展的`sext .. to`指令。 

零扩展就是最简单的，直接在高位补0，而符号扩展则是用原数的符号位来填充。也就是说我们如 下的代码：

```
%zext_integer = zext i8 -1 to i32 ; Extend 8 bit 0xFF to 32 bit 0x000000FF, get 255 %sext_integer = sext i8 -1 to i32 ; Extend 8 bit 0xFF to 32 bit 0xFFFFFFFF, get -
```

类似地，浮点型的数和整型的数也可以相互转换，使用`fptoui .. to`, `fptosi .. to`, `uitofp .. to` , `sitofp .. to`可以分别将浮点数转换为无符号、有符号整型，将无符号、有符号整型转换为 浮点数。不过有一点要注意的是，如果将大数转换为小的数，那么并不保证截断，如将浮点型的 257.1 转换成i8（上限为128），那么就会产生未定义行为。所以，在浮点型和整型相互转换的 时候，需要在高级语言层面做一些调整，如使用饱和转换等，具体方案可以看Rust最近1.45.0的更 新Announcing Rust 1.45.0和GitHub上的PR：Out of range float to int conversions using as has been defined as a saturating conversion.。

#### 指针类型

LLVM IR中的指针类型就是ptr。与C语言不同，LLVM IR中的指针不含有其指向内容的类型，也 就是说，类似于C语言中的void * 。我们之前提到，LLVM IR中的全局变量和栈上分配的变量都是 指针，所以其类型都是指针类型。 

在高级语言中，直接操作裸指针的机会都比较少，除非在性能极其敏感的场景下，由最厉害的大佬 才能操作裸指针。这是因为，裸指针极其危险，稍有不慎就会出现段错误等致命错误，所以我们使 用指针时应该慎之又慎。 

LLVM IR为大佬们提供了操作裸指针的一些指令。在C语言中，我们会遇到这种场景：

```
int x, y; 
size_t address_of_x = (size_t)&x; 
size_t address_of_y = address_of_x - sizeof(int); 
int also_y = *(int *)address_of_y;
```

这种场景比较无脑，但确实是合理的，需要将指针看作一个具体的数值进行加减。到x86_64的汇 编语言层次，取地址就变成了lea命令，解引用倒是比较正常，就是一个简单的mov。 

在LLVM IR层次，为了使指针能像整型一样加减，提供了`ptrtoint .. to`指令和`inttoptr .. to`指令，分别解决将指针转换为整型，和将整型转换为指针的功能。也就是说，我们可以粗略地将上 面的程序转写为

```
%x = alloca i32 ; %x is of type ptr, which is the address of variable x 
%y = alloca i32 ; %y is of type ptr, which is the address of variable y %address_of_x = ptrtoint ptr %x to i64 
%address_of_y = sub i64 %address_of_x, 4 
%also_y = inttoptr i64 %address_of_y to ptr ; %also_y is of type ptr, which is the address of variable y
```

#### 聚合类型

比起指针类型而言，更重要的是聚合类型。我们在 C 语言中常见的聚合类型有数组和结构体， LLVM IR也为我们提供了相应的支持。 

数组类型很简单，我们要声明一个类似C语言中的int a[4]，只需要

```
%a = alloca [4 x i32]
```

也就是说，C语言中的`int[4]`类型在LLVM IR中可以写成`[4 x i32]`。注意，这里面是个`x`不是`*`。 

我们也可以使用类似地语法进行初始化：

```
@global_array = global [4 x i32] [i32 0, i32 1, i32 2, i32 3]
```

特别地，我们知道，字符串在底层可以看作字符组成的数组，所以LLVM IR为我们提供了语法糖：

```
@global_string = global [12 x i8] c"Hello world\00"
```

在字符串中，转义字符必须以`\xy`的形式出现，其中`xy`是这个转义字符的ASCII码。比如说，字符 串的结尾，C语言中的`\0`，在LLVM IR中就表现为`\00`。

- 结构体

结构体的类型也相对比较简单，在C语言中的结构体

```
struct MyStruct { 
	int x; 
	char y; 
}; 
```

在LLVM IR中就成了

```
 %MyStruct = type { 
	 i32, 
	 i8 
}
```

我们初始化一个结构体也很简单：

```
@global_structure = global %MyStruct { i32 1, i8 0 } 
; or 
@global_structure = global { i32, i8 } { i32 1, i8 0 }
```

值得注意的是，无论是数组还是结构体，其作为全局变量或栈上变量，依然是指针，也就是说，`@global_array`的类型是`ptr`，`@global_structure`的类型也是`ptr`。接下来的问题就是，我们如何对聚合类型进行操作呢？

在 LLVM IR 中，如果我们相对一个聚合类型的某些字段进行操作，需要区分这个聚合类型是指针形式的，也就是以全局变量或者栈形式存储，还是值形式的，也就是以寄存器形式存储。

- `getelementptr`

首先，我们将介绍以指针形式存储的聚合类型，该如何访问其字段。

- 访问数组元素字段

我们先来看一个最直观的例子：

```cpp
struct MyStruct{
	int x;
	int y;
};

void foo(struct MyStruct *my_structs_ptr){
	int my_y = my_structs_ptr[2].y;
}
```

我们有一个`foo`函数，其接收了一个参数`my_structs_ptr`。从函数体的语义可知，这里这个参数，实际上指向了一个数组，我们要取这个数组的第三个元素的`y`字段。

我们先直接看结论，用 LLVM IR 来表示为

```cpp
%MyStruct = type{i32,i32}

define void @foo(ptr %my_structs_ptr){
	%my_y_in_stack = alloca i32 
	%my_y_ptr = getelementptr %MyStruct, ptr %my_structs_ptr, i64 2, i32 1
	%my_y_val = load i32, ptr %my_y_ptr 
	store i32 %my_y_val, ptr %my_y_in_stack 
	ret void
}
```

我们可以注意到，最核心的就是`getelementptr`指令了。它的四个参数的语义分别为

- `MyStruct`

我们要取地址的指针，它指向区域的类型为`%MyStruct`

- `ptr %my_structs_ptr`

我们要操作的指针，是`ptr %my_structs_ptr`

- `i64 2`

取偏移量为 2 的那个元素，也就是`my_structs_ptr[2]`

- i32 1

对于获得到的那个元素，取索引为 1 的字段，也就是`my_structs_ptr[2].y`

通过这个指令，我们获得了`my_structs_ptr[2].y`的地址，随后的 LLVM IR 指令就是将这个地址的值放到了局部变量中。

- 访问指针字段

接下来，我们看这样一个例子：

```cpp
struct MyStruct{
	int x;
	int y;
};

void foo(struct MyStruct *my_structs_ptr){
	int my_y = my_structs_ptr->y;
}
```

其对应的 LLVM IR 为

```cpp
%MyStruct = type { i32, i32 } 

define void @foo(ptr %my_structs_ptr) { 
	%my_y_in_stack = alloca i32 
	%my_y_ptr = getelementptr %MyStruct, ptr %my_structs_ptr, i64 0, i32 1 
	%my_y_val = load i32, ptr %my_y_ptr 
	store i32 %my_y_val, ptr %my_y_in_stack 
	ret void 
}
```

唯一的改动，就是将之前的偏移量`i64 2`改为`i64 0`。

这看上去挺符号直觉的。等等，符合直觉吗？

我们发现，即使是将`my_structs_ptr`看作是指向结构体的指针，而非指向数组的指针，仍然要加一个偏移量 0。这是因为，C语言中，对于一个数组`array`，`&array[0]`和指向首元素的`array_ptr`是同一个东西。为了兼容C语言这个特性，LLVM IR 在`getelementptr`中，将所有的指针都看作一个指向数组首地址的指针。因此，我们需要额外加一个`i64 0`的偏移量来解决这个问题。

- 级联访问

此外，`getelementptr`还可以接多个参数，类似于级联调用。我有C程序：

```cpp
struct MyStruct{
	int x;
	int y[5];
};
struct MyStruct my_structs[4];
```

那么如果我们想获得`my_structs[2].y[3]`的地址，只需要

```cpp
%MyStruct = type{
	i32;
	[5 x i32]
}
%my_structs = alloca [4 x %MyStruct]

%1 = getelementptr %MyStruct,ptr %my_structs,i64 2,i32 1,i64 3
```

- `extractvalue`和`insertvalue`

除了我们上面讲的这种情况，也就是把结构体分配在栈或者全局变量，然后操作其指针以外，

```cpp
;extract_insert_value.ll
%MyStruct = type{
	i32,
	i32
}
@my_struct = global %MyStruct { i32 1, i32 2 } 

define i32 @main() { 
	%1 = load %MyStruct, ptr @my_struct 
	
	ret i32 0 
}
```

这时，我们的结构体是直接放在虚拟寄存器`%1`里，`%1`并不是存储`@my_struct`的指针，而是直接 存储这个结构体的值。这时，我们并不能用`getelementptr`来操作`%1`，因为这个指令需要的是一 个指针。因此，LLVM IR提供了`extractvalue`和`insertvalue`指令。

因此，如果要获得`@my_struct`第二个字段的值，我们需要

```cpp
%2 = extractvalue %MyStruct %1, 1
```

这里的 1 就代表第二个字段（从0开始）。

类似地，如果要将`%1`的第二个字段赋值为 233，只需要

```
%3 = insertvalue %MyStruct %1,i32 233,1
```

然后`%3`就会是`%1`讲第二个字段赋值为 233 后的值。

`extractvalue`和`insertvalue`并不只适用于结构体，也同样适用于存储在虚拟寄存器中的数组，这里不再赘述。

- 标签类型

在汇编语言中，一切的控制语句、函数调用都是由标签来控制的，在 LLVM IR 中，控制语句也是需要标签来完成的。

- 元数据类型

在我们使用 Clang 将 C语言程序输出成 LLVM IR 时，会发现代码的最后几行有

```
!llvm.module.flags = !{!0, !1, !2, !3} 
!llvm.ident = !{!4} 

!0 = !{i32 1, !"wchar_size", i32 4} 
!1 = !{i32 8, !"PIC Level", i32 2} 
!2 = !{i32 7, !"PIE Level", i32 2} 
!3 = !{i32 7, !"uwtable", i32 2} 
!4 = !{!"Homebrew clang version 16.0.6"}
```

类似于这样的东西。在 LLVM IR 中，以`!`开头的标识符为元数据。元数据是为了将额外的信息附加在程序中传递给 LLVM 后端，使后端能够好地优化或生成代码。用于 Debug 的信息就是通过元数据形式传递的。我们可以使用 -g 选项：

```
clang -S -emit-llvm -g test.c
```

来在 llvm ir 中附加额外的 Debug 信息。关于元数据，在后续的章节里会有更具体的介绍。

- 属性

最后，还有一种叫做属性的概念。属性并不是类型，其一般用于函数。比如说，告诉编译器这个函数不会抛出错误，不需要某些优化等等。我们可以看到

```
define void @foo() nounwind {
	;...
}
```

这里nounwind 就是一个属性。 

有时候，一个函数的属性会特别特别多，并且有多个函数都有相同的属性。那么，就会有大量重复 的篇幅用来给每一个函数说明属性。因此，LLVM IR引入了属性组的概念，我们在将一个简单的C 程序编译成LLVM IR时，会发现代码中有

```
attributes #0 = { noinline nounwind optnone ssp uwtable "correctly-rounded divide-sqrt-fp-math"="false" "darwin-stkchk-strong-link" "disable-tail calls"="false" "frame-pointer"="all" "less-precise-fpmad"="false" "min-legal vector-width"="0" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "probe-stack"="___chkstk_darwin" "stack-protector-buffer-size"="8" "target cpu"="penryn" "target features"="+cx16,+cx8,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }
```

这种一大长串的，就是属性组。属性组总是以#开头。当我们函数需要它的时候，只需要

```
define void @foo #0 {
	;...
}
```

直接使用#0即可。关于属性，后续也会有专门的章节进行介绍。

#### 静态单分配（SSA）

- 每个变量只分配一次

- 每个变量在使用前都会被定义

```c
int factorial(int val){
	int temp = 1;
	for (int i = 2; i <= val; i++)
		temp *= i;
	return temp;
}
```

Pnis to the rescue

```
result = phi <ty> [<val0> <label0>], [<vall>, <label1>]
```

根据先前执行的基本块选择一个值

Allocas 为函数作用域分配内存

全局变量以静态方式填充模块的角色

它们总是指针，就像Allocas 返回的值一样

全局变量：

- 名称以`@`为前缀
- 必须具有类型
- 必须进行初始化
- 使用`global`关键字：`@gv = global i8 42`
- 或使用`constant`（表示不可被修改）：`@gv = constant i8 42`

总是指针

```
@gv = global i16 46

%load = load i16, i16* @gv
store i16 0, i16* @gv
```


#### 聚合类型：数组

聚合类型数组：

- a constant size
- an element type
- [for GVs] an ir initializer：`@array = global [17 x i8] zeroinitializer` 

The Get Element Pointer（GEP）instruction：

- Provides a way to calculate pointer offsets
- Abstracts away details like:
  - size of types
  - padding inside structs
- Intuitive to use
  - once you understand a few basic principles.

Manipulatin pointers

The Get Element Pointer（GEP）instruction：

```
<result> = getelementptr <ty>, <ty>* <ptrval>, [i32 <idx>]
```

```
%new_ptr = getelementptr i32,i32* %base, i32 1
```

GEP fundamentals

1. Understand the first index：
   1. It does NOT change the resulting pointer type
   2. It offsets by the base type.
2. Further indices:
   1. Offset inside aggregate types.
   2. Change the pointer type by removing one layer of aggregation

```
@a_gv = global [6 x i8] zeroinitializer
%new_ptr = getelementptr [6 x i8],[6 x i8]* @a_gv,i32 0
```

```
@a_gv = global [6 x i8] zeroinitializer
%new_ptr = getelementptr [6 x i8],[6 x i8]* @a_gv,i32 0
%elem_pt = getelementptr [6 x i8],[6 x i8]* @a_gv,i32 0,i32 0
```


#### 聚合类型：结构

Defined by

- A name
- Keyword type
- A list of types

GEPS with structs

```
%MyStruct = type {i8, i32, [3 x i32]}
@a_gv = global %MyStruct {i8 99, i32 17, [3 x i32] [i32 1, i32 2, i32 3]}
%new_ptr = getelementptr %MyStruct*,%MyStruct* @a_gv,i32 1
```

struct indices must be constants

```
@gv1 = global [4 x i8]
```

4. GEPs never load form memory

```

```


#### 虚拟寄存器

虚拟寄存器（Virtual Registers）

Those are local variables

Two flavors of names:

- Unnamed：`%<number>`
- Named：`%<name>`

LLVM IR has infnite registers

```
declare i32 @factorial(i32)

define i32 @main(){
	%1 = call i32 @factorial(i32 2)
	%2 = mul i32 %1,7
	%3 = icmp eq i32 %2,42
	%result = zext i1 %3 to i32
	ret i32 %result
}
```

Type，types everywhere

The instructions explicitly dictate the types expected.

Easy to figure out argument types

Easy to figure out return types


没有隐式转换。

要检查这是否是有效的 IR。

```
declare i32 @factorial(i32)

define i32 @main(){
	%1 = call i32 @factorial(i32 2)
	%2 = mul i32 %1,7
	%3 = icmp eq i32 %2,42
	ret i32 %3
}
```

The LangRef is your friend

call instruction

```
%! = call i32 @factorial(i32 2)
```


### 控制流

在汇编语言中跳转和函数没有太大的区别，然而到了 LLVM IR 中，这两者就有了比较大的区别。

在程序分析领域，往往会强调一对概念：数据流与控制流。所谓数据流，就是指一个程序中的数据，从硬盘到内存，从内存到寄存器，等等一系列的数据搬运、处理的过程。

而控制流，则是指程序执行指令的顺序。最简单的，我们的程序在除了顺序执行指令，还可以通过`if`语句进行条件跳转，通过`for`、`while`语句进行循环，还可以通过函数调用进入到别的函数。凡此种种，都是程序控制流的变化。

在使用 LLVM 作为编译器的时候，控制流往往就意味着更多的优化可能，如分支不仅、函数内联。在使用 LLVM 作为静态分析工具的过程中，控制流也意味着更高的复杂度，如间接跳转、间接调用的识别和恢复。

#### 控制语句

在控制流中，最基础的就是控制语句，也就是与汇编中的跳转相关的。

- 汇编层面的控制语句

在大多数语言中，常见的控制语句主要有四种：

- `if...else`
- `for`
- `while`
- `switch`

在汇编语言层面，控制语句则被分解为两种核心的指令：条件跳转与无条件跳转（`switch`其实还有一些工作，之后会提到）。我们下面分别来看看在汇编层面是怎样实现控制语句的。

`if...else`

我们有如下C 代码

```c
if (a > b) { 
	// Do something A 
} else { 
	// Do something B 
} 
// Do something C
```

为了将这个指令改写成汇编指令，我们同时需要条件跳转与无条件跳转。我们用伪代码表示其汇编 指令为：

```cpp
	Compare a and b 
	Jump to label B if comparison is a is not greater than b // conditional 
jump 
label A: 
	Do something A 
	Jump to label C // unconditional jump 
label B: 
	Do something B 
label C: 
	Do something C
```

汇编语言通过条件跳转、无条件跳转和三个标签（label A标签实际上没有作用，只不过让代码 更加清晰）实现了高级语言层面的if .. else语句。

`for`

我们有以下C代码：

```
for (int i = 0; i < 4; i++) { 
	// Do something A 
} 
// Do something B
```

为了将这个指令改写为汇编指令，我们同样地需要跳转与无条件跳转：

```cpp
	int i = 0 
label start: 
	Compare i and 4 
	Jump to label B if comparison is i is not less than 4 // conditional jump 
label A: 
	Do something A i++ 
	Jump to label start // unconditional jump label B: Do something B
```

而`while`与`for`则极其类似，只不过少了初始化与自增的操作，这里不再赘述。

根据我们在汇编语言中积累的经验，我们得出，要实现大多数高级语言的控制语句，我们需要四个东西：

- 标签
- 无条件跳转
- 比较大小的指令
- 条件跳转

- LLVM IR 层面的控制语句

下面就以我们上面的for循环的C语言版本为例，解释如何写其对应的LLVM IR语句。 

首先，我们对应的LLVM IR的基本框架为

```
%i = alloca i32 ; int i = ... 
store i32 0, ptr %i ; ... = 0 
%i_value = load i32, ptr %i 
; Do something A 
%1 = add i32 %i_value, 1 ; ... = i + 1 
store i32 %1, ptr %i ; i = ... 
; Do something B
```

这个程序缺少了一些必要的步骤，而我们之后会将其慢慢补上。

- 标签

在LLVM IR中，标签与汇编语言的标签一致，也是以:结尾作标记。我们依照之前写的汇编语言的 伪代码，给这个程序加上标签：

```
	%i = alloca i32                   ; int i = ... 
	store i32 0, ptr %i               ; ... = 0 
start: 
	%i_value = load i32, ptr %i 
A: 
	; Do something A 
	%1 = add i32 %i_value, 1          ; ... = i + 1 
	store i32 %1, ptr %i              ; i = ... 
B: 
	; Do something B
```

- 比较指令

LLVM IR提供的比较指令为icmp。其接受三个参数：比较方案以及两个比较参数。这样讲比较抽 象，我们就来看一下一个最简单的比较指令的例子：

```cpp
%comparison_result = icmp uge i32 %a, %b
```

这个例子转换为 C++ 语言就是

```cpp
bool comparison_result = ((unsigned int)a >= (unsigned int)b);
```

这里，`uge`比较方案，`%a`和`%b`就是用来比较的两个数，而`icmp`则返回一个`i1`类型的值，也就是 C++ 中的值`bool`值，用来表示结果是否为真。

`icmp`支持的比较方案很广泛：

- 首先，最简单的是`eq`与`ne`，分别代表不等或不相等。
- 然后，是无符号的比较`ugt`，`uge`，`ult`，`ule`，分别代表大于、大于等于、小于、小于等于。我们之前在数的表示中提到，LLVM IR 中一个整型变量本身的符号是没有意义的，而是需要看其在参与的指令中被看作是什么符号。每个方案的`u`就代表以无符号的形式进行比较。
- 最后，是有符号的比较`sgt`、`sge`、`slt`、`sle`，分别是其无符号版本的有符号对应。

加上比较指令之后，我们的例子就变成了：

```
	%i = alloca i32       ; int i = ... 
	store i32 0, ptr %i   ; ... = 0 
start: 
	%i_value = load i32, ptr %i %comparison_result = icmp slt i32 %i_value, 4 ; Test if i < 4 A: 
	; Do something A 
	%1 = add i32 %i_value, 1        ; ... = i + 1 
	store i32 %1, ptr %i            ; i = ... B: 
	; Do something B
```

- 条件跳转

在比较完之后，我们需要条件跳转。我们来看一下我们此刻的目的：若`%comparison_result`是`true`，那么跳转到`A`，否则跳转到`B`。

LLVM IR 为我们提供的条件跳转指令是`br`，其接受三个参赛，第一个参数是`i1`类型的值，用于作判断；第二和第三个参数分别为`true`和`false`时需要跳转到的标签。

比如，在我们的例子中，就应该是

```cpp
br i1 %comparison_result, label %A, label %B
```

我们把它加入我们的例子：

```
  %i = alloca i32         ;int i = ... 
  store i32 0, ptr %i     ; ... = 0 
start: 
	%i_value = load i32, ptr %i 
	%comparison_result = icmp slt i32 %i_value, 4 ; Test if i < 4 
	br i1 %comparison_result, label %A, label %B A: 
	; Do something A 
	%1 = add i32 %i_value, 1 ; ... = i + 1 
	store i32 %1, ptr %i ; i = ... 
B: 
	; Do something B
```

- 无条件跳转

无条件跳转更好理解，直接跳转到某一标签处。在 LLVM IR 中，我们同样可以使用`br`进行无条件跳转。如，我们要直接跳转到`start`标签处，则可以

```
br label %start
```

我们也把这个加入我们的例子：

```
	%i = alloca i32 ; int i = ... store i32 0, ptr %i ; ... = 0 
start: 
	%i_value = load i32, ptr %i 
	%comparison_result = icmp slt i32 %i_value, 4 ; Test if i < 4 
	br i1 %comparison_result, label %A, label %B 
A: 
	; Do something A 
	%1 = add i32 %i_value, 1 ; ... = i + 1 
	store i32 %1, ptr %i ; i = ... 
	br label %start 
B: 
	; Do something B
```

这样看上去就结束了，然而如果大家把这个代码交给`llc`的话，并不能编译通过，这是为什么呢？

- Basic block

一个函数由许多基本块组成。每个基本块包含：开头的标签、一系列标签、结尾是终结指令。一个基本块没有标签时，会自动赋给它一个标签。

所谓终结指令，就是指改变顺序的指令，如跳转、返回等。

我们来看看我们之前写好的程序是不是符号这个规定。`start`开头的基本块，在一系列指令后，以：

```
br i1 %comparison_result,label %A,label %B
```

结尾，是一个终结指令，`A`开头的基本块，在一系列指令后，以

```
br label %start
```

结尾，也是一个终结指令。`B`开头的基本块，在最后总归是需要函数返回的，所以也一定会带有一个终结指令。

一个基本块是可以没有名字的，所以，实际上还有一个基本块没有考虑到，就是函数开头的：

```
%i = alloca i32     ;int i = ...
store i32 0,ptr %i  ;... = 0
```

这个基本块，它并没有以终结指令结尾！

所以，我们把一个终结指令补充在这个基本块的结尾：

```
%i = alloca i32
store i32 0, ptr %i
br label %start
```

这样就完成了我们的例子，大家可以在本系列的 Github 的仓库中查看对应的代码`for.ll`。

- 可视化

LLVM 的工具链甚至为我们提供了可视化控制语句的方法。我们使用之前提到的 LLVM 工具链中用于优化的`opt`工具：

```
opt -p dot-cfg for.ll
```

然后会生成一个`.main.dot`的文件，如果我们在计算机上装有 Graphviz，那么就可以用

```
dot .main.dot -Tpng -o for.png
```

生成其可视化的控制流图（CFG）

- switch

下面我们来讲讲`switch`语句。我们有如下程序：

```cpp
int x;
switch (x){
	case 0:
		break;
	case 1:
		break;
	default:
		break;
}
```

我们先直接看看其转换成 LLVM IR 是什么样子的：

```cpp
switch i32 %x, label %C [ 
	i32 0, label %A 
	i32 1, label %B 
] 
A: 
	; Do something A 
	br label %end 
B: 
	; Do something B 
	br label %end 
C: 
	; Do something C 
	br label %end end: 
	; Do something else
```

其核心就是第一行的`switch`指令。其第一个参数`i32 %x`是用来判断的，也就是我们C语言中的`x`。第二个参数`label %C`是C语言中的`default`分支，这是必须要有的参数。也就是说，我们的`switch`必须要有`default`来处理。接下来是一个数组，其意义已经很显然了，如果`%x`值是`0`，就去`label %A`，如果值是1，就去`label %B`。

LLVM后端对switch语句具体到汇编层面的实现则通常有两种方案：用一系列条件语句或跳转表。 

一系列条件语句的实现方式最简单，用伪代码来表示的话就是 

```cpp
if (x == 0) { 
	Jump to label %A 
} else if (x == 1) { 
	Jump to label %B 
} else { 
	Jump to label %C 
}
```

这是十分符合常理的。然而，我们需要注意到，如果这个switch语句一共有n个分支，那么其查 找效率实际上是O(n)。那么，这种实现方案下的switch语句仅仅是if .. else的语法糖，除了增 加可维护性，并不会优化什么性能。 

跳转表则是一个可以优化性能的switch语句实现方案，其伪代码为： 

```cpp
labels = [label %A,label %B]
if(x<0 || x>1){
	Jump to label %C
}else{
	Jump to labels[x]
}
```

这只是一个极其粗糙的近似的实现，我们需要的是理解其基本思想。跳转表的思想就是利用内存中 数组的索引是O(1)复杂度的，所以我们可以根据目前的x值去查找应该跳转到哪一个地址，这就是 跳转表的基本思想。

根据目标平台和switch语句的分支数，LLVM后端会自动选择不同的实现方式去实现switch语 句。

- select

我们经常会遇到一种情况，某一变量的值需要根据条件进行赋值，比如说以下C语言的函数：

```cpp
void foo(int x){
	int y;
	if(x>0){
		y=1;	
	}else{
		y=2;
	}%
}
```

如果`x`大于0，则`y`为1，否则`y`为2。这一情况很常见，然而在C语言中，如果要实现这种功能，`y`需要被实现为可变变量，但实际上无论`x`如何取值，`y`只会被赋值一次，并不应该是可变的。

我们知道，LLVM IR 中由于SSA的限制，局部可变变量都必须分配在栈上，虽然 LLVM 后端最终会进行一定的优化，但写起代码来还需要冗长的`alloca`、`load`、`store`等语句。如果我们按照C语言的思路来写 LLVM IR，那么就会是：

```cpp
define void @foo(i32 %x){
	%y = alloca i32
	%1 = icmp sgt i32 %x,0
	br i1 %1,label %bture,label %bfalse
bture:
	store i32 1,ptr %y
	br label %end
bfalse:
	store i32 2,ptr %y
	br label %end
end:
	ret void
}
```

我们来看看其编译出的汇编语言是怎样的：

```
foo: 
# %bb.0: 
	cmpl  $0,%edi
	jle   .LBB0_2
# %bb.1: 
	movl  $1, -4(%rsp)
	jmp   .LBB0_3:
.LBB0_2: 
	movl   $2, -4(%rsp)
.LBB0_3
	retq
```

算上注释，C语言代码9行，汇编代码11行，LLVM IR 代码14行。这 LLVM IR 同时比低层次和高层次的代码都长，而且这种模式在真实的代码中出现的次数会非常多，这显然是不可用接受的。究其原因，就是这里把`y`看成了可变变量。那么，有没有什么办法让`y`不可变但仍然能实现这个功能呢？

我们来看看同样区分可变变量与不可变变量的 Rust 是怎么做的：

```rust
fn foo(x: i32){
	let y = if x>0 {1} else{2}
}
```

让代码简短的方式很简单，把`y`看作不可变变量，但同时需要语言支持把`if`语句视作表达式，当`x`大于0时，这个表达式返回 1，否则返回 2。这样，就很简单地实现了我们的需求。

LLVM IR 中同样也有这样的指令，那就是`select`，我们来吧上面的例子用`select`改写：

```cpp
define void @foo(i32 %x){
	%result = icmp sgt i32 %x,0
	%y = select i1 %result, i32 1, i32 2
}
```

`select`指令接受三个参数。第一个参数是用来判断的布尔值，也就是`i1`类型的`icmp`判断的结果，如果其为`true`，则返回第二个参数，否则返回第三个参数。极其合理。

`select`不仅可以简化 LLVM 代码，也可以优化生成的二进制程序。在大部分情况下，在AMD64架构中，LLVM 会将`select`指令编译为`CMOV CC`指令，也就是条件赋值，从而优化了生成的二进制代码

- phi

`select`只能支持两个选择，`true`选择一个分支，`false`选择另一个分支，我们是不是可以有支持多种选择的类似`switch`的版本呢？同时，我们也可以换个角度思考，`select`是根据`i1`的值来进行判断，我们其实可以根据控制流进行判断。这就是在 SSA 技术中大名鼎鼎的 phi 指令。

为了方便起见，我们首先先来看用 phi 指令实现的我们上面这个代码：

```cpp
define void @foo(i32 %x){
	%result = icmp sgt i32 %x,0
	br i1 %result, label %btrue, label %bfalse
btrue:
	br label %end
bfalse:
	br label %end
end:
	%y = phi i32 [1,%btrue],[t,%bfalse]
	ret void
}
```

我们看到，phi 的第一个参数是一个类型，这个类型表示其返回类型为 i32。接下来则是两个数组，其表示，如果当前的 basic block 执行的时候，前一个 basic block 是`%btrue`，那么返回 1，如果前一个 basic block 是`%bfalse`，那么返回 2。

也就是说，`select`是根据其第一个参数`i1`类型的变量的值来决定返回哪个值，而`phi`则是根据其之前是哪个 basic block 来决定其返回值。此外，phi 之后可以跟无数的分支，如`phi i32 [1,%a]`，`[2,%b]`，`[3,%c]`等，从而可以支持多分支的赋值。

#### 函数

定义与声明

- 函数定义

在 LLVM 中，一个最基本的函数定义的样子我们之前已经遇到过多次，就是`@main`函数的样子：

```
define i32 @main(){
	ret i32 0
}
```

在函数名之前可以加上参数列表，如：

```
define i32 @foo(i32 %a,i64 %b){
	ret i32 0
}
```

一个函数定义最基本的框架，就是返回值（i32）+ 函数名（@foo）+参数列表（（i32 %a，i64 %b））+函数体（{ret i32 0}）。

我们可以看到，函数的名称和全局变量一样，都是以`@`开头的。并且，如果我们查看符号表的话，也会发现其和全局变量一样进入了符号表。因此，函数也有和全局变量完全一致的 Linkage Types和Visibility Style，来控制函数名在符号表中的出现情况，因此，可以出现如

```
define private i32 @foo(){
	;...
}
```

这样的修饰符。

- 属性

此外，我们还可以在参数列表之后加上属性，也就是控制优化器和代码生成器的指令。如果我们单纯编译一个简短的C代码：

```
void foo(){}
```

其编译出的 LLVM IR 实际上是

```
define dso_local void @foo() #0{
	ret void
}
```

```
attributes #0 = { noinline nounwind optnone uwtable "frame-pointer"="all" "min legal-vector-width"="0" "no-trapping-math"="true" "stack-protector-buffer size"="8" "target-cpu"="x86-64" "target features"="+cx8,+fxsr,+mmx,+sse,+sse2,+x87" "tune-cpu"="generic" }
```

这里的#0就是一个属性组，其包含了noinline、nounwind等若干个函数的属性。这些属性可以 控制LLVM在优化和生成函数时的行为。大部分的属性可以在Function Attributes一节看到。

当函数的属性比较少时，我们可以直接把属性写在函数定义后面，而不用以属性的形式来写。例如下面这两种写法都是对的：

```cpp
define void @foo() nounwind {ret void}
; or
define void @foo() #0 {ret void}
attributes #0{
	nounwind ;...
}
```

我们知道，无论是在代码编译还是在程序分析的过程中，我们最常处理的都在函数级别。因此，属 性在这一过程中就是一个非常关键的概念。我们在编译器前端分析的过程中，遇到了特定的函数， 给它加上相应的属性；在编译器后端生成代码时，则判断当前函数是否有相应的属性，从而可以在 编译器前后端之间传递信息。

- 函数声明

除了函数定义之外，还有一种情况十分常见，那就是函数声明。我们在一个编译单元（模块）下， 可以使用别的模块的函数，这时候就需要在本模块先声明这个函数，才能保证编译时不出错，从而 在链接时正确将声明的函数与别的模块下其定义进行链接。

函数声明也相对比较简单，就是使用`declare`关键词替换`define`：

```
declare i32 @printf(i8*,...) #1 
```

这个就是在 C 代码中调用`stdio.h`库的`printf`函数时，在 LLVM IR 代码中可以看到的函数声明，其中`#1`就是又一大串属性组成的属性组。

- 函数的调用

在 LLVM IR 中，函数的调用与高级语言几乎没有什么区别：

```
define i32 @foo(i32 %a){
	;...
}

define void @bar(){
	%1 = call i32 @foo(i32 1)
}
```

使用`call`指令可以像高级语言那样直接调用函数。我们来仔细分析一下这里做了哪几件事：传递参数、执行函数、获得返回值

- 执行函数

我们知道，如果一个函数没有任何参数，返回值也是void类型，也就是说在C语言下这个函数是

```
void foo(void) { 
	// ... 
}
```

那么调用这个函数就没有了传递参数和获得返回值这两件事，只剩下执行函数，而这是一个最简单 的事，以AMD64架构为例：

1. 把函数返回地址压栈
2. 跳转到对应函数的地址

函数返回也是一个简单是事：

1. 弹栈获得函数返回地址
2. 跳转到相应的返回地址

- 传递参数与获得返回值

谈到这两点，就不得不说调用约定了。我们知道，在汇编语言中，是没有参数传递和返回值的概念 的，有的仅仅是让当前的控制流跳转到指定函数执行。所以，一切的参数传递和返回值都需要我们 人为约定。也就是说，我们需要约定两件事：

- 被调用的函数知道参数是放在哪里的
- 调用者希望知道调用函数的返回值是放在哪里的

这就是调用约定。不同的调用约定会产生不同的特效，也就产生了许多高级语言的feature。

**C调用约定**

最广泛使用的调用约定是 C 调用约定，也就是各个操作系统的标准库使用的调用约定。在 AMD64 架构下，C 调用约定是 System V 版本的，所有参数按顺序放入指定寄存器，如果寄存器不够，剩余的则从右往左顺序压栈。而返回值则是按先后顺序放入寄存器或者放入调用者分配的空间中，如果只有一个返回值，那么就会放在`rax`里。

在 LLVM IR 中，函数的调用默认使用 C 调用约定。为了验证，我们可以写一个简单的程序：

```
; calling_convention_test.ll 
%ReturnType = type { i32, i32 } 
define %ReturnType @foo(i32 %a1, i32 %a2, i32 %a3, i32 %a4, i32 %a5, i32 %a6, i32 %a7, i32 %a8) { ret %ReturnType { i32 1, i32 2 } } define i32 @main() { %1 = call %ReturnType @foo(i32 1, i32 2, i32 3, i32 4, i32 5, i32 6, i32 7, i32 8) ret i32 0 }
```

**fastcc**


可视化

#### 基本块

基本块（Basic Blocks）

- Branch - `br`
- Return - `ret`

```
define i32 @factorial(i32 %val){
	%is_base_case = icmp eq i32 %val,0
	br i1 %is_base_case,label %base_case,label %recursive_case
base_case:
	ret i32 1
recursive_case:
	%1 = add i32 -1, %val
	%2 = call i32 @factorial(i32 %0)
	%3 = mul i32 %val, %1
	ret i32 %2
}
```

calling function

- another Basic Blocks


#### CFG

执行流程图

控制流图CFG：

```shell
opt -analyze -dot-cfg-only test.ll
opt -analyze -dot-cfg test.ll

dot -Tpng xxx.dot -o 1.png
sz 1.png
```

```
define i32 @factorial(i32 %val){
	%is_base_case = icmp eq i32 %val,0
	br i1 %is_base_case,label %base_case,label %recursive_case
base_case:       ; preds = %entry
	ret i32 1
recursive_case:  ; preds = %entry
	%1 = add i32 -1, %val
	%2 = call i32 @factorial(i32 %0)
	%3 = mul i32 %val, %1
	ret i32 %2
}
```

```
opt -analyze -dot-cfg-only input.ll
```

- -dot-cfg=Generate.dot files.

Every Basic Block has a label

```
define i32 @factorial(i32 %val){
	%0: ;Implicit label
	%is_base_case = icmp eq i32 %val,0
	br i1 %is_base_case,label %base_case,label %recursive_case
base_case:       
	ret i32 1
recursive_case:
	%1 = add i32 -1, %val
	%2 = call i32 @factorial(i32 %0)
	%3 = mul i32 %val, %1
	ret i32 %2
}
```


## LLVM Pass

### Pass 介绍

Pass 用于处理代码编译中的中间代码，将中间代码经过 Pass 处理后编译成目标文件。

![](assets/LLVM.drawio.png)

### Pass 分类

- ModulePass

模块级 Pass（ModulePass）作用于整个 LLVM 模块，它可以跨函数甚至跨全局数据进行分析和转换。这使得 ModulePass 能够做一些全局性的优化或检查，例如死代码消除、内联优化、全局变量优化等。

当优化或分析需要全局视角（例如跨函数分析、跨模块优化、函数内联决策等）时，通常选择实现 ModulePass。例如，LLVM 内部的许多 IPO（过程间优化）都是通过模块级 Pass 来实现的。

- FunctionPass

函数级 Pass（FunctionPass）只针对单个函数内部进行处理，即每次仅处理模块中的一个函数。这种 Pass 适用于局部优化，如局部常量传播、局部死代码消除、指令合并等。

对于那些仅需要在函数内部进行局部转换或分析的优化，使用函数级 Pass 是比较理想的选择。例如，LLVM 内部常见的优化如指令选择、局部 SSA 转换、简单的算术优化等大多以 FunctionPass 形式实现。

大部分混淆变换 Pass 类型是 Function Pass。

- LoopPass

循环级 Pass

- CGSCCPass




### Pass 编程模型

接下来我们通过项目中的 Hello Pass 来学习 Pass 的基本结构，学习如何编写我们自己的 Pass。

LLVM18 中使用了新的 Pass Manger，所以我们需要使用新的pass Manger 来编写Pass。

- 头文件

```cpp
//包含 LLVM 旧版pass管理器的定义
#include "llvm/IR/LegacyPassManager.h"
//引入 LLVM 的新pass管理器的构建工具
#include "llvm/Passes/PassBuilder.h"
//包含定义用于插件化pass的接口
#include "llvm/Passes/PassPlugin.h"
//提供LLVM的输出流支持
#include "llvm/Support/raw_ostream.h"
```

要实现一个 Function Pass 只需要继承 FunctionPass 类，并实现`run`方法。LLVM Pass 机制在编译阶段对每一个函数调用`run`，这使得 FunctionPass 能够处理每一个函数。

`run`传入的参数是`llvm::Function`对象，这个对象包含了待分析函数的全部信息包括局部变量、基本块以及所有的指令。

可以这样理解，Function 由 BasicBlock 组成，BasicBlock 由 Instruction 组成。

例如遍历 Function 中的所有基本块代码如下

```cpp
for (BasicBlock &BB : F){
	//...
}
```

同样，也可以对基本块中的指令遍历

```cpp
for(Instruction &I : BB){
  //对指令 I 进行操作
}
```

### LLVM API

在 LLVM 中，llvm:Value 是一个抽象基类，代表 IR 中所有具有 值 属性的对象。简言之，所有可以出现在指令操作数位置的数据（无论是常量、变量、函数、指针、结构体等）都可看作是一种值，因此都继承自 llvm:Value。它定义了许多通用的接口，如获取该值的类型、名称、是否为常量、是否具有内部链接性等。

在 LLVM 的设计中，利用单一的基类 llvm:Value 帮助统一管理各种 IR 实体，使得在很多情况下，算法或 Pass 处理数据时都可以以 llvm:Value 为统一入口，而无需关注具体对象的种类。这种设计不仅使代码更加简洁，还大大降低了对不同 IR 对象进行操作时的复杂性。

LLVM 中的许多重要类都是从 llvm:Value 继承而来，包括但不限于：

- **llvm::Type**：虽然类型系统在一定程度上独立于值，但在某些场合也可以视为值的描述信息。部分 API 会返回与 llvm::Value 相关的类型信息。

- **llvm::Constant**：代表常量值，其子类如 llvm::ConstantInt、llvm::ConstantFP、llvm::UndefValue 等都描述了常量。

- **llvm::Function**：表示函数，在 LLVM IR 中函数也是一种值（它们有名称，可以被引用），因此 llvm::Function 继承自 llvm::Value。

- **llvm::GlobalVariable**：全局变量也属于值，并通过 llvm::GlobalVariable 表示。

- **llvm::Instruction**：每条 IR 指令也是一个值；它们计算出一个结果（在 SSA 形式下，每个指令产生一个唯一的、不可变的值）。

### LLVM Type

LLVM 中的一切数据都具有严格的类型信息，这对生成正确、类型安全的 IR 至关重要。**llvm::Type** 就是这一切类型的基类，它提供了一个统一的接口，让编译器能够查询和操作类型信息。无论是整数、浮点数、指针、数组、结构体还是函数类型，都可以通过 llvm::Type 来描述。

#### 基础类型的创建

一些基础类型的 Type 对象，可以用 Builder 来直接创建。

| Type   | `llvm::Type*`​          | Explanation                      |
| ------ | ---------------------- | -------------------------------- |
| void   | Builder.getVoidTy()    | just a void type                 |
| int    | Builder.getInt32Ty()   | assume 32 bit integers           |
| bool   | Builder.getInt1Ty()    | a one bit integer                |
| string | Builder.getInt8PtrTy() | pointer to array of bytes (int8) |

#### 类型检查与转换

LLVM 提供了一系列类型查询和转换函数，如 `isa<>`​、`cast<>`​ 和 `dyn_cast<>`​，用于判断一个 llvm::Type 指针是否属于某个特定子类，以及进行安全转换。例如：

```
llvm::Type *ty = ...;  
if (llvm::isa<llvm::IntegerType>(ty)) {  
    llvm::IntegerType *intTy = llvm::cast<llvm::IntegerType>(ty);  
    // 处理整数类型  
}
```

### LLVM Constant

LLVM Constant 的子类涵盖了各种类型的常量，主要包括：数值常量、指针和空常量以及复合常量等。在 llvm ir api 中常用 `llvm::Constant*`​ 是所有常量类型的基类。

例如创建一个整数常量的代码如下

```
ConstantInt::get(context, APInt(bitWidth, value));
```


### LLVM IR Builder

在 LLVM 中，**IRBuilder** 是一个用于简化生成中间表示（Intermediate Representation, IR）的辅助类。它提供了一个统一的 API，方便开发者在指定的插入点创建和插入指令，而无需直接操作底层的 LLVM API。

**主要功能：**

- **插入点管理：** IRBuilder 跟踪当前指令的插入位置，可以是基本块的末尾或特定的指令之前。通过设置插入点，后续生成的指令会被插入到该位置。

- **指令生成：** 提供了丰富的方法来生成各种类型的指令，包括算术运算、逻辑运算、控制流指令（如条件分支、跳转）和内存操作（如加载和存储）。

- **上下文管理：** IRBuilder 与 LLVM 上下文关联，确保所有生成的 IR 都在同一个上下文中，这对于内存管理和类型一致性至关重要。

代码例子：

```cpp
IRBuilder<> Builder(Context);  
// 创建基本块并设置插入点  
BasicBlock *BB = BasicBlock::Create(Context, "entry", Func);  
Builder.SetInsertPoint(BB);  
​  
// 创建返回常量 42 的指令  
Value *RetVal = Builder.getInt32(42);  
Builder.CreateRet(RetVal);
```

‍

### LLVM Function

LLVM 中每个函数都由一个 llvm::Function 对象表示。它既可以表示只包含函数原型的外部声明（declaration），也可以表示完整的函数定义（definition，包括函数体）。

具体来说，llvm::Function 具备以下几个特点：

- **全局对象**：llvm::Function 继承自 llvm::GlobalObject（以及 llvm::GlobalValue），这意味着函数是模块级（Module-level）的全局实体，具有独立的名称、链接类型和属性，可以被其他模块引用或链接。

- **继承自 llvm::Value**：作为 llvm::Value 的子类，函数具有类型信息（其类型总是一个指向函数类型的指针）和 use-def 链（用于优化和数据流分析）。

- **容纳基本块**：在函数定义中，函数内部由多个基本块（llvm::BasicBlock）组成，每个基本块是一串顺序执行的指令。函数对象提供了迭代器接口来遍历其包含的基本块，这对于许多优化 Pass 来说十分重要。

通过 llvm::Function，我们可以获取和设置函数的调用约定、函数属性（如 noinline、nounwind、optnone 等）以及链接类型（如 ExternalLinkage、InternalLinkage 等）。这些信息在代码生成和优化阶段都起到了关键作用。

‍

### LLVM BasicBlock

在 LLVM 的中间表示（IR）中，**基本块（BasicBlock）** 是函数的基本组成单元。

每个基本块是一系列顺序执行的指令序列，具有单一的入口和出口。

基本块之间通过控制流指令（如条件跳转和无条件跳转）连接，形成控制流图（CFG），描述程序的执行路径。

**基本块的主要特点：**

- **唯一入口和出口：** 每个基本块从特定的入口开始，按顺序执行其中的指令，直到遇到终止指令（如 `return`​、`br`​ 等）为止。

- **指令容器：** 基本块是指令的容器，包含了程序执行过程中的操作指令。

在 LLVM 的 C++ API 中，可以使用 `llvm::BasicBlock`​ 类来创建和操作基本块。

以下是一个简单的示例，展示如何创建一个基本块并将其添加到函数中：

```cpp
#include "llvm/IR/BasicBlock.h"  
#include "llvm/IR/Function.h"  
#include "llvm/IR/IRBuilder.h"  
#include "llvm/IR/LLVMContext.h"  
#include "llvm/IR/Module.h"  
#include "llvm/IR/Verifier.h"  
​  
using namespace llvm;  
​  
int main() {  
    LLVMContext context;  
    Module* module = new Module("MyModule", context);  
​  
    // 创建返回类型为 void，无参数的函数类型  
    FunctionType* funcType = FunctionType::get(Type::getVoidTy(context), false);  
    // 创建函数  
    Function* function = Function::Create(funcType, Function::ExternalLinkage, "MyFunction", module);  
​  
    // 创建基本块，并将其添加到函数中  
    BasicBlock* entry = BasicBlock::Create(context, "entry", function);  
​  
    // 使用 IRBuilder 在基本块中插入指令  
    IRBuilder<> builder(entry);  
    builder.CreateRetVoid();  
​  
    // 验证函数的正确性  
    verifyFunction(*function);  
​  
    // 输出模块的 IR  
    module->print(outs(), nullptr);  
​  
    delete module;  
    return 0;  
}
```

‍

上述代码创建了一个名为 `MyFunction`​ 的函数，并在其中添加了一个名为 `entry`​ 的基本块。在该基本块中，使用 `IRBuilder`​ 插入了一条返回指令。最后，验证函数的正确性并输出生成的 IR。

‍

**基本块的遍历：**

在 LLVM 中，可以通过函数对象遍历其包含的基本块。例如，给定一个函数 `F`​，可以使用以下代码遍历其中的基本块：

```
for (BasicBlock &BB : F) {  
    // 对基本块 BB 进行操作  
}
```

同样地，可以遍历基本块中的指令：

```
for (Instruction &I : BB) {  
    // 对指令 I 进行操作  
}
```

这种遍历方式对于分析和优化 Pass 非常常见。

**基本块的前驱和后继：**

在控制流图中，基本块之间通过前驱（predecessors）和后继（successors）关系连接。可以使用以下方法获取基本块的前驱和后继：

```cpp
// 获取前驱基本块  
for (BasicBlock *Pred : predecessors(&BB)) {  
    // 对前驱基本块 Pred 进行操作  
}  
​  
// 获取后继基本块  
for (BasicBlock *Succ : successors(&BB)) {  
    // 对后继基本块 Succ 进行操作  
}

```

这些函数在遍历控制流图时非常有用。

**基本块的插入和删除：**

可以使用 `BasicBlock`​ 类的成员函数来插入和删除基本块。例如，使用 `splitBasicBlock`​ 可以将一个基本块拆分为两个，使用 `eraseFromParent`​ 可以将基本块从所属的函数中移除并删除。

需要注意的是，操作基本块时应确保控制流的正确性，避免生成不合法的 IR。


### LLVM Instruction

在 LLVM 的中间表示（Intermediate Representation, IR）中，**指令（Instruction）** 是构成程序逻辑的基本单元。每条指令表示一个操作，如算术运算、内存访问或控制流操作。在 LLVM 的 C++ API 中，`llvm::Instruction`​ 类是所有指令的基类，提供了统一的接口来处理不同类型的指令。

**​`llvm::Instruction`​**​ **类的主要特点：**

- **继承关系：** `llvm::Instruction`​ 继承自 `llvm::User`​，而 `llvm::User`​ 又继承自 `llvm::Value`​。这意味着每条指令既是一个值（可以被其他指令引用），也是一个用户（可以使用其他值作为操作数）。

- **操作码（Opcode）：** 每条指令都有一个操作码，表示指令的类型。可以使用 `getOpcode()`​ 方法获取指令的操作码，并通过 `Instruction::Add`​、`Instruction::Sub`​ 等枚举值进行比较。

- **操作数：** 指令可以有多个操作数，这些操作数是指令所操作的值。可以使用 `getOperand()`​ 方法获取特定的操作数。

- **基本块：** 每条指令都属于一个基本块（`llvm::BasicBlock`​），可以通过 `getParent()`​ 方法获取所属的基本块。

‍

**常见的指令类型：**

`llvm::Instruction`​ 类有多个子类，每个子类表示特定类型的指令。例如：

- **算术指令：** 如 `llvm::BinaryOperator`​，用于表示加法、减法等二元运算。

- **内存指令：** 如 `llvm::LoadInst`​ 和 `llvm::StoreInst`​，用于表示内存的加载和存储操作。

- **控制流指令：** 如 `llvm::BranchInst`​ 和 `llvm::ReturnInst`​，用于表示分支和返回操作。


以下是一个使用 LLVM C++ API 创建加法指令的示例：

```cpp
#include "llvm/IR/IRBuilder.h"  
#include "llvm/IR/LLVMContext.h"  
#include "llvm/IR/Module.h"  
#include "llvm/IR/Function.h"  
#include "llvm/IR/BasicBlock.h"  
​  
using namespace llvm;  
​  
int main() {  
    LLVMContext Context;  
    Module *M = new Module("my_module", Context);  
    IRBuilder<> Builder(Context);  
​  
    // 创建函数签名  
    FunctionType *FuncType = FunctionType::get(Builder.getInt32Ty(), false);  
    Function *Func = Function::Create(FuncType, Function::ExternalLinkage, "my_function", M);  
​  
    // 创建基本块  
    BasicBlock *BB = BasicBlock::Create(Context, "entry", Func);  
    Builder.SetInsertPoint(BB);  
​  
    // 创建常量值  
    Value *LHS = Builder.getInt32(10);  
    Value *RHS = Builder.getInt32(20);  
​  
    // 创建加法指令  
    Value *Sum = Builder.CreateAdd(LHS, RHS, "sum");  
​  
    // 创建返回指令  
    Builder.CreateRet(Sum);  
​  
    // 输出 IR  
    M->print(outs(), nullptr);  
​  
    delete M;  
    return 0;  
}  
```


‍

比如在控制流平坦化混淆中，最常见的是获取基本块尾部的结尾指令并判断指令的类型。

```
if (!isa<BranchInst>(BB.getTerminator()) && !isa<ReturnInst>(BB.getTerminator()))  
    return;
```

如果基本块结尾的指令(`BB.getTerminator()`​) 不是跳转分支指令，并且也不是返回指令，则不混淆当前函数。

llvm api 中常用 `isa<X>(Y)`​ 来判断 Y 是不是 X 类型的一个实例。


```cpp
using namespace llvm;

//将所有代码放在匿名空间中，避免与其他代码冲突
namespace {

// 该方法实现了该 pass 的功能。
//访问器函数
//visitor函数：该函数被调用以打印每个函数的名称和参数数量
void visitor(Function &F) {
    errs() << "(llvm-tutor) Hello from: "<< F.getName() << "\n";
    errs() << "(llvm-tutor)   number of arguments: " << F.arg_size() << "\n";
}


// 新的 PM 实现
//pass实现
//HelloWorld结构体: 实现了 PassInfoMixin，包含了 run 方法，执行对每个函数的分析。
//isRequired: 此函数确保即使函数被标记为 optnone，该pass仍会被执行。

struct HelloWorld : PassInfoMixin<HelloWorld> {
  // 主入口点，接受要运行该 Pass 的 IR 单元（&F）和相应的 Pass 管理器（如果需要可以查询）。
  PreservedAnalyses run(Function &F, FunctionAnalysisManager &) {
    visitor(F);
    return PreservedAnalyses::all();
  }

	// 如果 isRequired 返回 false，那么这个 pass 将会被跳过，针对带有 `optnone` LLVM 属性的函数。请注意，`clang -O0` 会将所有函数标记为 `optnone`。
  static bool isRequired() { return true; }
};
} // namespace

//-----------------------------------------------------------------------------
// 新的 PM 注册
//-----------------------------------------------------------------------------

//pass注册
//插件信息注册：定义了如何注册pass的插件信息。使用 registerPipelineParsingCallback 注册 hello-world 名称的pass
llvm::PassPluginLibraryInfo getHelloWorldPluginInfo() {
  return {LLVM_PLUGIN_API_VERSION, "HelloWorld", LLVM_VERSION_STRING,

	//Lambda表达式
          [](PassBuilder &PB) {

		//注册回调
		//参数
			//StringRef Name: 命令行中传入的pass名称。
			//FunctionPassManager &FPM: 用于管理函数pass的工具。
			//ArrayRef<PassBuilder::PipelineElement>: 管道元素的数组，包含在命令行中指定的pass元素。
            PB.registerPipelineParsingCallback(
                [](StringRef Name, FunctionPassManager &FPM,
                   ArrayRef<PassBuilder::PipelineElement>) {
               
               //条件判断，检查传入的名称是否为hello-world，如果匹配则调用FPM.addPass将HelloWorld pass添加到功能pass 管理器中
                  if (Name == "hello-world") {
                    FPM.addPass(HelloWorld());
                    return true;
                  }
                  return false;
                });
          }};
}


```

- 入口函数

```cpp
// 这是 pass 插件的核心接口。它确保 opt 在命令行中通过 -passes=hello-world 添加到 pass 流水线时能够识别 HelloWorld。

//入口函数
//插件接口：这是LLVM加载pass时调用的接口，确保LLVM能够识别 HelloWorld pass
extern "C" LLVM_ATTRIBUTE_WEAK ::llvm::PassPluginLibraryInfo
llvmGetPassPluginInfo() {
  return getHelloWorldPluginInfo();
}
```

- 入口函数

- Pass注册

### 源码内编译

源码内编译我们需要将我们自己编写的 Pass 作为源码项目的一部分。

我们在`project/llvm/lib/Transforms/`下新建一个目录存放我们的 Pass。

然后我们需要在我们的 Pass 目录下新建一个cmake文件。

```
llvm
```

然后我们需要在上一级目录即`Transforms`下的`CMakeLists.txt`文件中添加一行，即我们自己编写的Pass。

```cmake
add_subdirectory(EncodeFunctionName)
```

然后在构建目录中重新执行`make`编译（不用删除已编译文件）。


可以从 LLVM 的源代码树中开发 LLVM Pass

```
<project dir>/
    |
    CMakeLists.txt
    <pass name>/
        |
        CMakeLists.txt
        Pass.cpp
        ...
```

内容 ：`<project dir>/CMakeLists.txt`

```cmake
find_package(LLVM REQUIRED CONFIG)

add_definitions(${LLVM_DEFINITIONS})
include_directories(${LLVM_INCLUDE_DIRS})

add_subdirectory(<pass name>)
```

内容 ：`<project dir>/<pass name>/CMakeLists.txt`

add_library(LLVMPassname MODULE Pass.cpp)

请注意，如果您打算将此通道合并到 LLVM 源代码树中，请在 将来，将 LLVM 的内部函数与 MODULE 参数一起使用可能更有意义，而不是通过...`add_llvm_library`

将以下内容添加到 （在`<project dir>/CMakeLists.txt``find_package(LLVM ...)`)

```cmake
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
include(AddLLVM)
```

然后更改为`<project dir>/<pass name>/CMakeLists.txt`

```cmake
add_llvm_library(LLVMPassname MODULE
  Pass.cpp
  )
```

完成 pass 的开发后，您可能希望将其集成 添加到 LLVM 源代码树中。您可以通过两个简单的步骤来实现它：

1. 正在将文件夹复制到目录中。`<pass name>``<LLVM root>/lib/Transform`
2. 向 中添加行。`add_subdirectory(<pass name>)``<LLVM root>/lib/Transform/CMakeLists.txt`



```cmake
list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
```

- **作用**：将 LLVM 的 CMake 目录添加到 `CMAKE_MODULE_PATH` 中，这样 CMake 可以找到与 LLVM 相关的模块。


```cmake
# 设置最低cmake版本
cmake_minimum_required(VERSION 3.20.0)

# 定义项目名称
project(SimpleProject)

# 查找 LLVM：使用 find_package 命令查找 LLVM。REQUIRED 表示如果找不到 LLVM，将报错并停止配置过程。CONFIG 表示使用 LLVM 的配置文件。
find_package(LLVM REQUIRED CONFIG)

# 打印找到的 LLVM 版本：输出找到的 LLVM 版本信息，${LLVM_PACKAGE_VERSION} 是由 find_package 设置的变量。
message(STATUS "Found LLVM ${LLVM_PACKAGE_VERSION}")

# 打印配置文件路径：输出正在使用的 LLVMConfig.cmake 的路径，${LLVM_DIR} 是包含该文件的目录。
message(STATUS "Using LLVMConfig.cmake in: ${LLVM_DIR}")

include_directories(${LLVM_INCLUDE_DIRS})

separate_arguments(LLVM_DEFINITIONS_LIST NATIVE_COMMAND ${LLVM_DEFINITIONS})

# 添加编译定义：将之前分离出的 LLVM 定义添加到编译器选项中，确保在编译时包含所需的宏定义。
add_definitions(${LLVM_DEFINITIONS_LIST})

# 创建可执行文件：定义一个可执行文件 simple-tool，其源代码为 tool.cpp。
add_executable(simple-tool tool.cpp)

# 映射 LLVM 组件到库名：使用 llvm_map_components_to_libnames 将指定的 LLVM 组件（support, core, irreader）映射到相应的库名，并存储在 llvm_libs 变量中。
llvm_map_components_to_libnames(llvm_libs support core irreader)

target_link_libraries(simple-tool ${llvm_libs})
# 链接库：将映射到的 LLVM 库链接到可执行文件 simple-tool，确保在编译时可以找到和使用这些库。
```

### 源码外编译

源码外编译分为两种方式，一种是命令行编译，一种是cmake编译。

#### 命令行编译

命令行编译是最简单的方法。

```bash
clang `llvm-config --cxxflags` -Wl,-znodelete -fno-rtti -fPIC -shared Hello.cpp -o LLVMHello.so `llvm-config --ldflags`
```

其中

- `llvm-config`提供了`CXXFLAGS`与`LDFLAGS`参数方便查找LLVM的头文件与库文件。 如果链接有问题，还可以用`llvm-config --libs`提供动态链接的LLVM库。 具体`llvm-config`打印了什么，请自行尝试或查找官方文档。
- `-fPIC -shared` 显然是编译动态库的必要参数。
- 因为LLVM没用到RTTI，所以用`-fno-rtti` 来让我们的Pass与之一致。
- `-Wl,-znodelete`主要是为了应对LLVM 5.0+中加载ModulePass引起segmentation fault的bug。如果你的Pass继承了ModulePass，还请务必加上。

现在，你手中应该有一份编译好的LLVMHello.so了。根据刚才阅读过的官方文档的介绍，你知道可以通过命令

```bash
$ clang -c -emit-llvm main.c -o main.bc # 随意写一个C代码并编译到bc格式
```

来使用它。

#### 使用cmake编译

项目结构

```

```

在 Pass 目录中编写 Pass 的 CMakeLists.txt

```cmake
cmake_minimum_required(VERSION 3.10)

# 项目
project(MyLLVMProject)

# 查找 LLVM 包。REQUIRED 表示如果找不到 LLVM，CMake会报错并停止构建。
# CONFIG 指示 CMake 查找 LLVM 的配置文件。
find_package(LLVM REQUIRED CONFIG)

# 将 LLVM 的头文件路径添加到编译器的搜索路径中。
include_directories(${LLVM_INCLUDE_DIRS})

# 创建一个名为 MyPass 的动态库。
# 并指定源文件，MODULE 表示该库不会生成可执行文件，而是用于动态加载的插件。
add_library(MyPass MODULE my_pass.cpp)

# 将 MyPass 库与 LLVM 库链接。
target_link_libraries(MyPass PRIVATE ${LLVM_LIBRARIES})
```

### 运行Pass

#### 使用opt运行Pass

```shell
opt -load-pass-plugin ./LLVMHello.so -passes=hello-world demo.bc -o /dev/null
```

#### 使用Clang运行Pass

```shell
clang -Xclang -load -Xclang /path/to/libMyPlugin.so -Xclang -plugin -Xclang static-call-counter test.c -o test
```

### 将Pass注册到Clang

#### 自动加载Pass到clang

首先，假设你已经编写并编译了一个 LLVM Pass，目标是将这个 Pass 集成到 `clang` 编译流程中，避免每次手动通过 `opt` 运行。为了实现这个目标，可以使用 `-Xclang` 参数将 Pass 加载到 `clang` 中。

例如，假设你已经有一个编译好的 `LLVMHello.so` 插件，那么可以通过以下命令将这个 Pass 加载到 `clang`：

```shell
clang -Xclang -load -Xclang path/to/LLVMHello.so main.c -o main
```

#### 使用环境变量自动化编译

如果你在开发过程中使用了 `autotools` 或类似的构建系统，并且想要在每次编译时自动加载 Pass，可以通过设置 `CC` 和 `CFLAGS` 来传递 Pass 插件参数。通过以下命令，你可以将 Pass 插件集成到构建过程中：

```shell
$ CC=clang CFLAGS="-Xclang -load -Xclang path/to/LLVMHello.so" ./configure
$ make
```

#### 编写clang Wrapper脚本

如果你觉得每次手动输入 `-Xclang` 参数比较繁琐，可以编写一个 `clang` 的 wrapper 脚本来自动处理这些参数。这个脚本将接收标准的 `clang` 参数，并将 Pass 插件的加载参数附加到命令中。

假设你创建一个名为 `hello-clang` 的脚本，内容如下：

```shell
#!/bin/bash
# hello-clang wrapper to add LLVM pass automatically

# 设置插件路径
PLUGIN_PATH="path/to/LLVMHello.so"

# 构造新的命令行参数
clang "$@" -Xclang -load -Xclang "$PLUGIN_PATH"
```

然后你就可以使用 `hello-clang` 命令来自动加载 Pass 插件：

```shell
$ CC=hello-clang ./configure && make
```


## 后言

LLVM 项目在二进制安全领域应用广泛，其对中间代码插桩的特性尤其为代码混淆和漏洞挖掘提供了关键技术支撑，催生了 OLLVM 等经典项目，类似思想也被应用于 AFL 等非 LLVM 系工具中。

## 参考

>[LLVM架构简介 - LLVM IR入门指南](https://evian-zhang.github.io/llvm-ir-tutorial/01-LLVM%E6%9E%B6%E6%9E%84%E7%AE%80%E4%BB%8B.html)
>[LLVM编译器入门（一）：LLVM整体设计](https://www.bilibili.com/video/BV18j411B7TF/?share_source=copy_web&vd_source=a1e0ec33522dd06cf97338beeab421c6)
>[基于LLVM-18，使用New Pass Manager，编写和使用Pass]( https://www.bilibili.com/video/BV1Bf3neVEnN/?share_source=copy_web&vd_source=a1e0ec33522dd06cf97338beeab421c6)
>[LLVM Pass入门导引](https://zhuanlan.zhihu.com/p/122522485)
>[LLVM基本概念入门](https://zhuanlan.zhihu.com/p/140462815)
>[​github.com/imdea-software/LLVM_Instrumentation_Pass/blob/master/InstrumentFunctions/Pass.cpp](https://link.zhihu.com/?target=https%3A//github.com/imdea-software/LLVM_Instrumentation_Pass/blob/master/InstrumentFunctions/Pass.cpp)
>[Helpful Hints for Common Operations](https://link.zhihu.com/?target=https%3A//llvm.org/docs/ProgrammersManual.html%23helpful-hints-for-common-operations)
>[ProgrammersManual - The Core LLVM Class Hierarchy Reference](https://link.zhihu.com/?target=https%3A//llvm.org/docs/ProgrammersManual.html%23the-core-llvm-class-hierarchy-reference)
>[banach-space/llvm-tutor: A collection of out-of-tree LLVM passes for teaching and learning](https://github.com/banach-space/llvm-tutor#)