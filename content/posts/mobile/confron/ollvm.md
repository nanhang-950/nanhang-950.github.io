---
title: OLLVM混淆和反混淆
date: 2025-11-30 23:30:38
categories: Mobile
tags: 
  - 加固开发
mathjax:
top:
---

## 前言



## 简介

OLLVM（Obfuscator-LLVM）是瑞士西部应用科学大学安全小组于 2010 年 6 月发起的一个开源项目，旨在基于 LLVM 构建一套通用的代码混淆工具，以提升软件在逆向工程面前的安全性。

作为 LLVM 的 IR 级混淆框架，OLLVM 的混淆操作直接作用于 LLVM IR，只要目标平台能够使用 LLVM 编译链，理论上都可以使用 OLLVM 生成混淆后的二进制。其核心机制是通过编写自定义 Pass 修改和扰动 IR，使后端生成的机器码继承相同的混淆效果。

OLLVM 提供了三类经典的混淆策略：

- **控制流平坦化（Control Flow Flattening）**
- **虚假控制流（Bogus Control Flow）**
- **指令替换（Instructions Substitution）**

不过现在在实际的生产环境中，更常使用的是 Hikari。Hikari 是 OLLVM 的改进版。与原版 OLLVM 相比，Hikari 增强和扩展了多个关键能力，包括：

- **常量加密混淆（Constant Encryption）**
- **强化的控制流平坦化（Enhanced CFF）**
- **更强的不透明谓词（Opaque Predicates）**
- **反调试、反 Hook 等运行时防护机制**

## 环境搭建

环境：Win11MSVC

需安装：

- CMake
- Ninja

### 下载源码

很可惜 Hikari 已经被删除了，所以我自己做了一个 Hikari-LLVM18 版。

```shell
git clone https://github.com/nanhang-950/Hikari-LLVM18
cd Hikari-LLVM18
git submodule update --init --recursive
```

### 编译项目

```shell
cd Hikari-LLVM15  
mkdir build  
cd build  
cmake -S ../llvm -B . -G "Ninja" ^
    -DCMAKE_BUILD_TYPE=Release ^
    -DLLVM_ENABLE_PROJECTS="clang" ^
    -DLLVM_INCLUDE_TESTS=OFF ^
    -DCMAKE_CXX_FLAGS="/utf-8" ^
    -DCMAKE_C_FLAGS="/utf-8"
ninja -j12

cmake ../llvm \
  -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS="clang" \
  -DLLVM_TARGETS_TO_BUILD="AArch64"

make -j$(nproc) 
```

### 整合NDK

llvm 18 选择`android-ndk-r27c`，将`build/bin`目录下的`clang.exe`、`clang++.exe`、`clang.cl.exe`替换到`ndk\27\toolchains/llvm/prebuilt/linux-x86_64/bin/clang-18`目录下。


### 混淆使用

在`build.gradle`中配置需要混淆的组件：

```gradle
defaultConfig {
    applicationId "com.example.myapplication"
    minSdk 24
    targetSdk 33
    versionCode 1
    versionName "1.0"
 
    testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
 
    externalNativeBuild {
        cmake {
            cppFlags "-mllvm -irobf-cff"
        }
    }
}
```

编译混淆样本：

```shell
CC=/home/pandaos/android-ndk/android-ndk-r21e/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android22-clang -I/home/pandaos/android-ndk/android-ndk-r21e/toolchains/llvm/prebuilt/linux-x86_64/lib64/clang/9.0.9/include 

$CC -mllvm -irobf-indbr main.c -o main-obf
```

ndk 里面的一些 headers 没有自动导入，需要手动处理导入处理一下。

跟ollvm类似,可通过编译选项开启相应混淆. 如启用间接跳转混淆

```shell
$ path_to_the/build/bin/clang -mllvm -irobf -mllvm --irobf-indbr test.c
```

对于使用autotools的工程

```shell
$ CC=path_to_the/build/bin/clang or CXX=path_to_the/build/bin/clang
$ CFLAGS+="-mllvm -irobf -mllvm --irobf-indbr" or CXXFLAGS+="-mllvm -irobf -mllvm --irobf-indbr" (or any other obfuscation-related flags)
$ ./configure
$ make
```

## Angr符号执行

<img src="./assets/Pasted image 20250326152447.png" alt="Pasted image 20250326152447" style="zoom: 33%;" />

### 符号执行原理

定义：符号执行（Symbolic Execution）是一种程序分析技术，通过将程序输入替换为符号变量，探索程序的所有可能路径，并生成路径约束条件进行求解，从而帮助发现漏洞、验证程序正确性或生成测试用例。

核心思想：通过将程序的输入抽象为符号变量，追踪这些变量在程序中的操作，并通过生成路径条件来验证各条路径的可行性。这种方法有助于检测潜在的错误和漏洞，并能够覆盖更多的执行路径，从而提高程序分析的全面性和准确性。

什么是符号执行，一般我们运行一个程序需要给程序输入内容，直接输入内容后程序执行。这属于传统执行的使用实际的具体值来运行程序，而符号执行使用符号值来运行程序。

符号执行的三个关键点：

- 变量符号化
- 程序执行模拟
- 约束求解

符号执行的关键在于：

- 符号化输入：将程序的输入替换为符号变量。
- 路径探索：模拟程序的执行过程，探索程序的所有可能路径，特别是在条件分支处，符号执行会分别生成两个路径来代表条件为真和条件为假的情况。
- 路径约束：每一条路径都会产生一个路径约束，路径约束是关于符号变量的数学公式，表示这条路径的条件。
- 约束求解：符号执行生成的路径条件可以通过约束求解器（例如 Z3）来求解，求解器可以找到满足这些条件的符号变量具体值，从而推导出实际输入。

### 符号化输入

符号执行首先要将程序的输入替换为符号变量，而是表示一个不确定的集合，即它们可以取多个可能的值。，符号变量不仅可以表示未知数，还可以用于表达任何一个可能的值。

比如在下例代码中，用户输入了两个整型`x`和`y`。`x`和`y`可能是任何值，即都是未知数。

- 例

```python
x=input("请输入x: ")
y=input("请输入y: ")

if x > y:
	print("success")
else:
	print("try again")
```

我们对这个程序进行符号执行，首先将`x`和`y`替换为符号变量。

由于`x`和`y`都是整型，所以我们要定义两个整型的符号变量

```python
from z3 import *

# 符号变量
x = Int('x') 
y = Int('y') 
```

### 路径探索

### 路径约束

然后生成路径条件，在上述例子中我们的路径条件是`x > y`。

所以我们通过`z3`向符号变量添加约束。

```python
# 创建求解器
s = Solver()

# 加入路径条件：x > y
s.add(x > y)

if s.check() == sat:
    model = s.model()
    print(f"Path 1: x = {model[x]}, y = {model[y]}")  # 满足 x > y
else:
    print("No solution for x > y")

# 创建新求解器实例，加入另一个路径条件：x <= y
s.push()
s.add(x <= y)
```

### 约束求解

使用求解器来求解可能的输入。

```python
# 求解路径条件
if s.check() == sat:
    model = s.model()
    print(f"Path 2: x = {model[x]}, y = {model[y]}")  # 满足 x <= y
```


### 面临的问题

- 程序的路径符号化：
  - 程序中的输入被抽象为符号变量
  - 每次条件分支都会生成新的路径约束
  - 程序执行不再是单一路径，而是**路径集合**
- 路径爆炸：
  - 每个条件分支 ⇒ 路径数指数增长
  - 循环、递归、状态机尤为严重
- 约束求解困难
  - SMT 约束复杂、非线性
  - 包含位运算、数组、指针、哈希
  - 求解时间不可预测

### 环境搭建

推荐用虚拟环境搭建：

```shell
python3 -m venv angr_env
source angr_env/bin/activate
pip install angr
```

### 基础概念

- `Project`：一个可加载的二进制文件在 Angr 中的表示形式；
- `State`：程序点的状态，包括寄存器、内存以及当前约束限制等；
- Block：基本块，但是 angr 中的基本块与 ida 基本块不同，angr 中 call 也算控制流图的边；
- 符号值：由一个未知的符号来代替运算数值，用于约束求解，符号值有符号名，比如 x，y，z；
- 具体值：一个具体的数值，比如 1，2，3。

| 名称                 | 描述                     | 传入值     |
| -------------------- | ------------------------ | ---------- |
| auto_load_libs       | 是否自动加载程序的依赖库 | True/False |
| skip_libs            | 希望避免加载的库         | 库名       |
| except_missling_libs | 无法解析库时是否抛出异常 | True/False |
| force_load_libs      | 强制加载的库             | 库名       |
| ld_path              | 共享库的优先搜索路径     | 路径名     |

使用 angr 时最重要的就是效率问题，少加载一些无关结果的库能够提升 angr 的效率，如下：

```python
p=angr.Project("test",auto_load_libs=False)
```

Project 类中有许多方法和属性，例如加载的文件名、架构、程序入口点，大小端等等。

```python
import angr

binary = '/bin/true'
p = angr.Project(binary, auto_load_libs=False)
  
print("文件:", p.filename)
print("入口点:", hex(p.entry))
print("架构:", p.arch)
print("位数:", p.arch.bits)
print("端序:", p.arch.memory_endness)
print("加载器:", p.loader)
```

`angr.Project`​ 函数能够加载 ELF、PE 文件，并返回 project 用于表示加载的可执行文件，后续的分析操作都在都需要在 project 里完成。

`auto_load_libs=False`​ 参数表示让 angr 不加载。


### 符号变量

Angr 使用 `claripy`​ 库来创建符号值。

```python
claripy.BVS('x', 32) # 创建一个 32 位的符号值
claripy.BVV(10, 32) # 创建一个 32 位的具体值，值为 10
```

angr 是一个混合符号执行框架，为了符号执行速度，不是所有的数据都以符号化的形式存在，如果一个值是确定的，那么就可以用确定的具体值参与执行，从而加速执行效率，`claripy.BVV`​ 用于创建一个具体值变量。

使用 BVS 和 BVV 创建的符号变量可以直接在 Python 进行运算，例如

```python
import claripy

x = claripy.BVS('x', 32)
y = claripy.BVV(10, 32)
print(x + 1)
print(x + y + 1)
```

运行将输出

```python
<BV32 x_0_32 + 0x1>
<BV32 x_0_32 + 0xb>
```

y 是一个具体值变量，因此可以直接参与运算，`10 + 1 == 0xb`​ 最终在第二条输出中显示的是 0xb。

### State状态

Project 实际上只是将二进制文件加载进来了，要执行它，实际上是对 SimState 对象进行操作，它是程序的状态。

要创建状态，需要使用 Project 对象中的 factory，它还可以用于创建模拟管理器和基本块，如下：

```python
state=p.factory.entry_state()
```

预设状态有四种方式如下：


| 预设状态方式       | 描述                                                     |
| ------------------ | -------------------------------------------------------- |
| entry_state        | 初始化状态为程序运行到程序入口点处的状态                 |
| blank_state(addr=) | 大多数数据都没有初始化，状态中下一条指令为 addr 处的指令 |
| full_init_state    | 共享库和预定义内容已经加载完毕，例如刚加载完共享库       |
| call_state         | 准备调用函数的状态                                       |


在 Angr 中，程序在某一个程序点的执行状态叫做 State，执行过程就是从一个 State 向另外一个 State 转换。每个 State 的寄存器、内存、约束限制等都是独立的，互相不影响。

状态包含了程序运行时的一切信息，寄存器、内存的值、文件系统以及符号变量等，这些信息的使用等用到时再进一步说明。

`entry_state`和`blank_state`是常用的两种方式，后者通常用于跳过一些极大降低 angr 效率的指令，它们间的对比如下：

```python
state=p.factory.entry_state()
print(state.regs.rax,state.regs.rip)

state=p.factory.blank_state(addr=0x400000)
```

‍
BV 即 BitVector 意思是位向量


默认情况下，state 所在的程序点位置的地址是基本块开始地址，angr 符号执行的“单步执行”是执行一个基本块的所有指令，并产生后继基本块的 state。


获取初始化 State，获取入口点 state（从入口点开始执行）

```python
proj = angr.Project('/bin/ls')
state = proj.factory.entry_state()
print(state)
```

程序将输出

```python
<SimState @ 0x406d30>
```

`/bin/ls`​ 程序的入口地址是 `0x406d30`​，当前 state 表示位于 `pc = 0x406d30`​ 地址，并且该地址的指令还未执行的状态。

其它创建 state 的方法还有

* `proj.factory.blank_state(addr=0x401234)`: 在 0x401234 位置创建 state
* `proj.factory.call_state(addr=0x401234, arg1, arg2)`​ ：创建一个调用函数的 state

访问 state 寄存器：

每个 state 的寄存器是独立的，angr 允许访问和修改 state 中的寄存器值。

注意，在 angr 符号执行框架中，寄存器的值可能是具体值，也可能是符号值，这一点跟模拟执行、动态执行不一样。

访问(修改) R0 寄存器: `state.regs.r0`​

访问(修改) RAX 寄存器: `state.regs.RAX`​

给 R0 寄存器设置一个符号值

```py
# 给寄存器赋值一个符号值
state.regs.rax = state.solver.BVS('symbolic_rax', 64)

# 给寄存器赋值一个具体值 123
state.regs.rdi = 123
```

当然也可以使用 `claripy.BVS`​ 创建符号值，`claripy.BVV`​ 创建具体值。

‍

通过寄存器名的字符串读取寄存器

```python
 state.regs.get('reg_name')
```

通过寄存器名的字符串修改寄存器

```py
setattr(state.regs, 'reg_name', value)
```

访问 state 内存：

在 angr 中，每一个 state 的内存状态是独立的，就像每个程序点都有一份内存快照一样。当 state 符号执行，创建新 state 的时候，新 state 的内存首先从原 state 复制一份，根据指令语义进行内存修改。这与传统的动态执行不一样，传统动态执行的内存是全局的。

`state.memory`​ 是访问 state 内存的主要接口。

修改内存

```py
address = 0x401234
size_in_bytes = 16
state.memory.store(address, state.solver.BVS('mem', 8 * size_in_bytes))
```

读取内存

```py
address = 0x401234
size_in_bytes = 16
data = state.memory.load(address, size_in_bytes)
```

### Simulation Manager

上述方式只是预设了程序开始分析时的状态，我们要分析程序就必须要让它到达下一个状态，这就需要模拟管理器的帮助（简称 SM）。

使用以下指令能创建一个 SM，它需要传入一个 state 或者 state 的列表作为参数：

```python
simgr = p.factory.simgr(state)
```

SM 中有许多列表，这些列表被称为 stash，它保存了处于某种状态的 state，stash有如下几种：


| stash         | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| active        | 保存接下来可以执行并且将要执行的状态                         |
| deadended     | 由于某些原因不能继续执行的状态，例如没有合法指令，或者有非法指针 |
| pruned        | 与 solve 的策略有关，当发现一个不可解的节点后，其后面所有的节点都优化掉放在 pruned 里 |
| unconstrained | 如果创建 SM 时启用了 save_unconstrained，则没有约束条件的 state 会放在这里 |
| unsat         | 如果创建 SM 时启用了 save_unsat，则被认为不可满足的 state 会放在这里 |

默认情况下，state 会被存放在 active 中。

可以通过 step() 非法来让处于 active 的 state 执行一个基本块，这种操作不会改变 state 本身（类比docker）：

```python
state=p.factory.entryi.entry_state()
simgr=p.factory.simgr(state)
print(state.regs.rax,state.regs.rip)

print(simgr.one_active)

simgr.step()
print(simgr.one_active)

print(state.regs.rax,state.regs.rip)
```

### Explorer

可以使用 explorer 方法去执行某个状态，直到直到目标指令或者 active 中没有状态为止，它有如下参数：

- find：传入目标指令的地址或地址列表，或者一个用于判断的函数
- avoid：传入要避免的指令的地址或地址列表，或者一个用于判断的函数，用于减少路径

angr 会沿着分支按照某种策略（默认DFS）进行状态搜索，当达到目标状态时，使用约束求解器（claripy）对收集到的约束进行求解。

因此，使用 angr 一般分为如下步骤：

- 创建 Project，预设 state
- 创建位向量和符号变量，保存在内存/寄存器/文件或其他地方
- 将 state 添加到 SM 中
- 运行，探索满足条件的路径
- 约束求解获取执行结果

### 符号化判断

判断一个 angr 中的表达式是否含有符号在反混淆程序分析、漏挖挖掘等场景中非常有意义。

基于符号执行运算特性，能够实现一个高精度的污点分析框架。

例如在某个程序点将 R0 寄存器设置赋值一个符号值，等程序遇到间接跳转时，判断 pc 寄存器是否是一个符号化的表达式，如果是则说明 R0 寄存器能够控制 PC 寄存器。

另外一个例子，将控制流平坦化混淆中的状态变量符号化，对函数进行符号执行，就能找出所有与状态变量相关的指令，这样能精准区分混淆产生的垃圾指令与真实代码指令。具体细节将在后续讨论。

```python
state.regs.rdi = 123
print(state.regs.rdi.symbolic) # 输出 False

state.regs.rdx = state.solver.BVS('symbolic_rdx', 64) + 1
print(state.regs.rdx.symbolic) # 输出 True
```

state 内存读取出来的数据也可以判断是否符号化

```python
address = 0x801234
size_in_bytes = 16
state.memory.store(address, state.solver.BVS('mem', 8 * size_in_bytes))
data = state.memory.load(address, size_in_bytes)
print(data) # <BV128 mem_46_128>
print(data.symbolic) # True
```

‍

### 执行 state

如何从一个 state ，通过符号执行，得到新的 state 呢？

最简单的方法是通过单步执行的形式，来得到当前 state 经过符号执行所产生的新的 state，只需要调用 `state.step()`​ 函数就能执行一个基本块的所有指令。

```python
import angr

proj = angr.Project('/bin/ls')
state = proj.factory.entry_state()

print("state 地址:", hex(state.addr))
successors = state.step().successors
print("后继 state(%d 个):" % len(successors))
for i, s in enumerate(successors):
    print("后续 state %d: addr: %s" % (i, hex(s.addr)))

# state 地址: 0x406d30
# 后继 state(1 个):
# 后续 state 0: addr: 0x62a200
```

在符号执行中，如果遇到条件分支，并且条件符号化，无法判断具体执行哪个分支的时候，就会产生两个新的 state ，分别对应满足条件的分支和不满足条件的分支。同时，与 state 绑定的约束限制也会增加对应的条件。

`state.step().successors`​ 调用引擎符号执行一次，返回后继 state 数组。

注意，即使 state 经过 step 函数符号执行一次后，它内部的状态仍然不会改变，新产生的 state 是在当前 state 的基础上复制产生的，换句话说，state 的中的寄存器、内存以及约束限制仍然是执行前的内容，就像程序快照一样。

### 延迟求解

angr 会判断路径条件是否满足，有时候这样很影响效率，我们可以设置 angr 使其找到 State 之后再进行求解

```python
state.options |= {angr.options.LAZY_SOLVES}
```

### 调用指定函数

如何通过 angr 调用一个已知地址的函数。

```python
import angr

proj = angr.Project("demo", auto_load_libs=False)

# 假设 target_func 的地址已知
func_addr = 0x401136

# 创建调用状态，传入参数 3 和 5
state = proj.factory.call_state(func_addr, 3, 5)

simgr = proj.factory.simgr(state)
simgr.run()

# 获取函数返回值
ret_state = simgr.deadended[0]
print("Return value:", ret_state.solver.eval(ret_state.regs.eax))
```

### 模拟执行

注意 angr 默认加载地址是 0x400000

```python
import angr

proj = angr.Project("demo", auto_load_libs=False)

							   #函数入口   #第一个参数 #第二个参数
state = proj.factory.call_state(0x401136, 1, 2) #用于从指定函数地址开始执行，模拟函数调用
simgr = proj.factory.simulation_manager(state)
simgr.run()

ret_state = simgr.deadended[0] #获取函数返回的状态
print(state.solver.eval(state.memory.load(0x42ED24, 19), cast_to=bytes))
```

### 更高效的 state 执行管理

`state.step()`​ 只能执行一个基本块，如果想连续执行，就需要手动添加循环，并且管理路径，十分麻烦。

为了解决这个问题，angr 中使用 Simulation Managers (simgr) 来管理 state，使用更加高级的路径执行策略、搜索策略等。

```python
state = proj.factory.entry_state()
simgr = proj.factory.simulation_manager(state)
```

使用 Project 提供的工厂函数` proj.factory.simulation_manager`​ 就能从一个 state 创建 simgr。

simgr 将管理的 state 按照不同类型分类，并存入对应的 stash 列表：

* `simgr.active`​ ：活跃 state 列表，simgr 每次单步执行，会执行 active 中所有的 state。

* `simgr.deadended`​：当状态因为某种原因无法继续执行时，它会被放入 `deadended`​，这种情况可能包括没有更多有效的指令、所有后继状态的不可满足性（unsat），或者无效的指令指针。
* `simgr.pruned`: 在使用 `LAZY_SOLVES`​ 时，状态不会被检查是否满足条件，除非绝对必要。当发现一个状态在 `LAZY_SOLVES`​ 模式下是不可满足的（unsat）时，会遍历状态的历史，找出最初变为不可满足的时刻。所有从这个时刻开始的后代状态（也会是不可满足的，因为状态不能变得满足）会被修剪并放入这个 stash。
* `simgr.unconstrained`: 如果在 `SimulationManager`​ 构造函数中提供了 `save_unconstrained`​ 选项，所有被判定为不受约束的状态（即指令指针由用户数据或其他符号数据控制的状态）会放入这个 stash。
* `simgr.unsat`: 如果在 `SimulationManager`​ 构造函数中提供了 `save_unsat`​ 选项，所有被判定为不可满足的状态（例如，约束相互矛盾，比如输入既要是 "AAAA" 又要是 "BBBB"）会放入这个 stash。

其中最重要的就是 active 列表，这个列表存放了活跃的 state，simgr 每次单步执行会单步执行 active 列表中的所有 state，并将新的 state 放入对应的 stash 列表。

举例来说，最初创建 simgr 时的 state 将放入 active 列表，此时 active 列表中只有初始 state。调用 `simgr.step`​ ，angr 遍历 active 列表，依次执行每一个 state 的 step 函数，并将新产生的 state 放入对应的 stash 列表，通常将放入 active 列表。

```python
import angr

proj = angr.Project('/bin/ls')
state = proj.factory.entry_state()

simgr = proj.factory.simulation_manager(state)

# 不断执行，直到找到一个符号化的条件分支
while len(simgr.active) == 1:
	print(simgr.active)
    simgr.step()
  
print(simgr.active)
```

上面这个例子中，从程序入口开始符号执行，直到找到第一个符号化分支。只需要在循环中判断 active 长度即可实现。

simgr 更加高级的用法是路径搜索，`simgr.explore(find=x)`​ 的作用就是在符号执行过程中搜索满足 find 参数条件的 state。find 可以传递一个地址或者带有state 参数的函数。​

上面例子中，要回答什么样的输入能够让程序输出“Good Job.”，需要找到一个从 main 函数入口到 “Good Job.” 的 state。

首先，在符号执行前将标准输入符号化（scanf 输入），从 main 函数入口的 state 开始符号执行，直到找到 “Good Job.” 的 state，该位置 state 中的约束限制由标准输入符号描述，使用该 state 求解满足所有约束限制的符号值，就能得到能让程序输出 “Good Job.” 的字符串。

## Z3

Z3 是一个微软出品的开源约束求解器，能够解决很多种情况下的给定部分约束条件寻求一组满足条件的解的问题。

Z3是由微软研究院开发的高性能定理证明器，专注于解决包含布尔逻辑、算术、位运算等复杂约束条件的问题。作为当前最先进的SMT（可满足性模理论）求解器之一，Z3广泛应用于软件验证、程序分析、网络安全和人工智能等领域。

angr 符号执行引擎基于 z3 约束求解实现，因此学习 z3 约束求解器对于理解 angr 工作原理至关重要。

安装：

```shell
pip install z3-solver
```

### 核心概念速览

1. **求解器对象**：创建`Solver()`​实例管理约束条件
2. **变量声明**：使用`Int()`​, `Bool()`​, `BitVec()`​等函数
3. **约束添加**：通过`add()`​方法添加条件
4. **求解验证**：`check()`​检查可满足性，`model()`​获取解

### 基本数据类型

- `Int`：整数类型；
- `Real`：实数类型；
- `Bool`：布尔类型；
- `Array`：数组类型；
- `BitVec('a',8)`：字节向量。

其特征 bitvec 可以是特定大小的数据类型，不一定是 8 位，例如 C语言中的 int 型开源用 bitvec('a',32)表示。

### 基本语句

- `Solver()`：创建一个通用求解器，以进行下一步的求解
- `add()`：添加约束条件，通常在 solver() 命令之后，添加的约束条件通常是一个逻辑等式。
- `check()`：添加完约束条件后，用来检测解的情况，有解的时候会返回 sat，无解的时候会返回 unsat
- `model()`：在存在解的时候，该函数会将每个限制条件所对应的解集的交集，进而得出正解。

分别对应设未知数，列方程组，判断方程是否有界，解方程组

在使用 z3 进行约束求解之前我们首先需要获得一个求解器类实例，本质上其实就是一组约束的集合：

```python
s=z3.Solver()
```

### 基本使用

假设有方程组：

```
30x + 15y = 675
12x + 5y = 265
```

使用 z3 来解这个方程组，首先是未知数

```python
from z3 import *

x = Int('x')
y = Int('y')
```

列方程

```python
s = Solver()
s.add(30*x+15*v==675)
```

### 变量表示

- Int-整数型

```python
from z3 import *
a=Int('a')#声明单个整型变量
a,b=Ints('a b')#声明多个整型变量
```

- Real-实数型

```python
from z3 import *
a=Real('a')#声明单个实性变量
a,b=Reals('a b')
```

- BitVec-向量

```python
from z3 import *
a=BitVec('a',8)#声明单个8位的变量
a,b=BitVec('a b',8)#声明多个8位的变量
```

只有向量类型可以进行位运算（异或，左右移位等）

- Bool

```python
p=z3.Bool(name='p')
```

- 整型宇实数类型变量之间可以互相进行转换

```python
>>> z3.ToReal(x) 
ToReal(x)
>>> z3.ToInt(y) 
ToInt(y)
```

### 常量表示

除了 python 原有的常量数据类型外，我们也可以使用`z3`自带的常量类型参与运算：

```python
>>> z3.IntVal(val = 114514) # integer 
114514 
>>> z3.RealVal(val = 1919810) # real number 
1919810 
>>> z3.BitVecVal(val = 1145141919810, bv = 32) # bit vector，自动截断 
2680619074 
>>> z3.BitVecVal(val = 1145141919810, bv = 64) # bit vector 
1145141919810
```

### 添加约束

我们可以通过求解器的`add()`方法为指定求解器添加约束条件，约束条件可以直接通过`z3`变量组成的式子进行表示：

```python
s.add(x*5==10)
s.add(y*1/2==x)
```

对于布尔类型的式子而言，我们可以使用z3内置的`And()`、`Or()`、`Not()`、`Implies()`等方法进行布尔逻辑运算：

```python
s.add(z3.Implies(p,q))
s.add(r==z3.Not(q))
s.add(z3.Or(z3.Not(p),r))
```

### 约束求解

当我们向求解器中添加约束条件之后，我们可以使用`check()`方法检查约束是否是可满足的（satisfiable，即z3是否能够帮我们找到一组解）：

- `z3.sat`：约束可以被满足
- `z3.unsat`：约束无法被满足

```python
s.check()
```

若约束可以被满足，则我们可以通过`model()`方法获取到一组解：

```python
s.model()
```

对于约束条件比较少的情况，我们也可以无需创建求解器，直接通过`solve()`方法进行求解：

```python
z3.solve(z3.Implies(p,q),r==z3.Not(q),z3.Or(z3.Not(p),r))
```

### 常规使用

- 解方程

```python
from z3 import *
a,b=Ints('a b')
#solve函数
solve(a+b==10,a+3*b==12)
```

- 化简表达式

```python
from z3 import *
x=Int('x')
y=Int('y')
print (simplify(x + y + 2*x + 3))
print (simplify(x < y + x + 2))
print (simplify(And(x + 1 >= 3, x**2 + x**2 + y**2 + 2 >= 5)))
```

- 当需要取出特定变量的时候，可以使用 Solver 求解器

```python
from z3 import *
a,b = Ints('a b')

s=Solver()  #创建一个解的对象s
s.add(a + b == 10)#添加约束条件
s.add(a + 3 * b == 12)

if s.check() == sat: #check() 检查解是否存在，存在会return 'sat'
	result = s.model() #输出

print(result)

print(result[a])
print(result[b])
```

## 污点分析

**污点分析（Taint Analysis）** 是一种广泛应用于软件安全和漏洞检测领域的程序分析技术。其核心思想是通过追踪程序中不可信或敏感数据（称为“污点数据”）的传播路径，识别这些数据是否可能被用于危险操作（如执行恶意代码、触发漏洞等）。通过这种方式，污点分析能够帮助发现潜在的安全漏洞、隐私泄露或逻辑缺陷。

‍污点分析工作原理：

1. **污点标记（Taint Source）** 在程序执行或静态分析过程中，将外部输入（如用户输入、网络数据、文件读取等）标记为“污点源”，即可能携带风险的初始数据。

2. **污点传播（Taint Propagation）** 跟踪污点数据在程序中的流动路径，包括变量赋值、函数调用、表达式计算等操作。若污点数据通过合法操作（如字符串拼接、算术运算）传播到其他变量，则新变量也会被标记为污点。

3. **敏感操作检测（Sink Check）** 当污点数据到达“敏感操作点”（如数据库查询、系统命令执行、内存写入等）时，分析器会触发警报，判定是否存在安全风险（如SQL注入、命令注入、缓冲区溢出等）。

## 反混淆与污点分析

在反混淆场景下的二进制分析任务中，经常需要区分正常指令和混淆产生的垃圾指令，我们可以借助污点分析来区分正常指令和非正常指令。

### 控制流平坦化

在反混淆场景下的二进制分析任务中，经常需要区分正常指令和混淆产生的垃圾指令，我们可以借助污点分析来区分正常指令和非正常指令。

<img src="assets/image-20250129220357-dy3vi2x.png" alt="image" width="60%" style="zoom:50%;" >

这样能够找出所有状态变量相关的指令，包括内存加载、寄存器移动、加减、比较等指令。

### 虚假控制流

如下图是一个 Hikari 虚假控制流（简称 bcf）效果。

![](assets/image-20250130144539-h8vodik.png)

bcf 混淆会生成许多虚假的基本块，并用一个永真式嵌入原程序，Hikari bcf 生成的表达式中依赖于一些全局变量。

用污点分析识别 bcf 相关的指令，首先将 bcf 涉及到的全局变量全部设置为污点源。Hikari bcf 的全局变量很容易识别，所有Hikari bcf 全局变量位于一个固定的全局连续区间，只需要人工标记区间的开始和结束位置即可。


## 基于 angr 构建污点分析框架

针对反混淆需求，污点分析框架应该有如下能力：

（1）某个指定地址的指令位置，该位置的指令是否会被污点源污染？

（2）某个指定地址的指令位置，指定的寄存器是否被污点源污染？

（3）某个指定地址的指令位置，指定的内存是否被污点源污染？

利用 angr 符号执行的特性可以构建一个基于符号执行的污点分析框架。“污点” 用 angr 中的符号值来表示，随着符号值参与执行运算，污点对应的符号值随着执行指令的语义污点传播。

同时，angr 的多指令集符号执行能力，让基于 angr 构建的污点分析框架能够跨指令集工作。

### 污点源

针对反混淆的需求，污点源通常是寄存器和内存。

```python
def taint_memory(state, address, size_in_bytes):  
    state.memory.store(address, claripy.BVS("taint", size_in_bytes*8))  

def taint_register(state, reg_name, reg_size_in_bytes=8):  
    setattr(state.regs, reg_name, claripy.BVS("taint", reg_size_in_bytes*8))
```

实现原理是将指定的内存或寄存器符号化，并且符号名是 `taint`​。

### 判断表达式是否污点

用表达式来抽象“寄存器”和“内存数据”，从寄存器和内存中读取出来的值都称为表达式。

```python
def taint_check_expr(expr):  
    """  
    check if an expression is tainted  
    """  
    if expr.symbolic:  
        for vname in list(expr.variables):  
            if 'taint' in vname:  
                return True  
    return False
```

接下来构建判断寄存器是否被污点感染的函数

```python
def taint_check_register(state, reg_name):  
    """  
    check if a register is tainted  
    """  
    
    try:  
        if reg_name == 'nzcv': # 架构相关！只在 aarch64 验证  
            reg_name = 'flags'  
        rs = state.regs.get(reg_name)  
    except AttributeError:  
        logger.warning("taint_check_register: %s not supported by angr" % reg_name)  
        return False  
    
    return taint_check_expr(rs)
```

### 污点传播(执行)

污点源设置后，用 angr 做符号执行就能实现污点传播，具体的执行策略，比如控制流之间的转移策略还需要根据不同的任务具体讨论。然而 angr 默认的指令执行策略不合适反混淆场景的污点传播（它一次性执行一个 block而不是一条指令），因此要构建一个适合反混淆分析场景的污点指令级执行策略。

angr 符号执行有一个特性，它一次性执行一个 block，很难满足 ”某个指定地址的指令位置“，特别是位于基本块中间的指令。经过查阅 angr 源码，发现 state 的 step 能够指定执行指令的数量，将指令数量设置为 1 即可实现逐指令插桩污点分析。当然还有一些其它比较 trick 的技巧可以实现逐指令执行，比如 angr inspect。

下代码能够执行一条指令。

```python
successors = proj.factory.successors(run_state, num_inst=1).successors
```

在如上代码的基础上，构建一个污点分析专用的 step 函数。

```python
def taint_step(proj, state, run_until_pc=None, num_inst=None, check_every_inst=True):  
    """  
    对程序状态执行污点传播步骤，返回后继状态及污染状态列表  

    Args:  
        proj (angr.Project): angr 项目对象  
        state (angr.SimState): 当前待分析的模拟程序状态  
        run_until_pc (int, optional): 目标程序计数器值，运行直到达到该PC地址时停止。  
                                      若为None则表示不启用此停止条件，默认为None  
        num_inst (int, optional): 指定要执行的指令数量。若为None则表示不启用此停止条件，  
                                  默认为None  
        check_every_inst (bool, optional): 是否在每条指令执行后检查污染状态。  
                                           若为False则仅在停止时检查，默认为True  
  
    Returns:  
        tuple: 包含两个列表的元组：  
            - successors (list[SimState]): 执行产生的所有后继状态列表  
            - tainted_list (list[SimState]): 被识别为受污染的状态列表  
  
    功能说明：  
        通过污点分析跟踪程序状态的传播过程。可根据PC地址或指令数量设置停止条件，  
        返回分析过程中产生的所有后继状态和检测到的受污染状态  
    """  
    
    tainted_list = []  
    regs_read_write_cache = {}  
    run_state = state.copy()  
    run_inst = 0  
    
    while True:  
        cur_pc = run_state.addr  
        if run_until_pc is not None and cur_pc == run_until_pc:  
            return [run_state], tainted_list  
        
        # 解析指令的读写寄存器  
        if check_every_inst and (cur_pc not in regs_read_write_cache):  
            for insn in run_state.block().capstone.insns: # block 返回一块的指令，提前缓存  
                read_regs, write_regs = insn.regs_access() # 使用 capstone 解析指令的读写寄存器  
                regs_read_write_cache[insn.address] = ([insn.reg_name(r) for r in read_regs], [insn.reg_name(r) for r in write_regs])  
        
        # 检查 tainted  
        if check_every_inst:  
            read_regs, write_regs = regs_read_write_cache[cur_pc]  
            for reg in read_regs:  
                if taint_check_register(run_state, reg):  
                    if cur_pc not in tainted_list:  
                        logger.info("tainted pc: %x" % cur_pc)  
                        tainted_list.append(cur_pc)  
        
        try:  
            successors = proj.factory.successors(run_state, num_inst=1).successors  
        except Exception as e:  
            logger.warning("taint_step: failed to execute %x: %s, skip" % (cur_pc, e))  
            temp_state = run_state.copy()  
            temp_state.regs.pc = cur_pc + 4  
            successors = [temp_state]  
        
        run_inst += 1  
        if num_inst is not None and run_inst >= num_inst:  
            return successors, tainted_list  
        
        if check_every_inst:  
            read_regs, write_regs = regs_read_write_cache[cur_pc]  
            for reg in write_regs:  
                for state in successors:  
                    if taint_check_register(state, reg):  
                        if cur_pc not in tainted_list:  
                            logger.info("tainted pc: %x" % cur_pc)  
                            tainted_list.append(cur_pc)  
​  
        if len(successors) != 1:  
            return successors, tainted_list  
        else:  
            run_state = successors[0]
```

每一条指令执行前，首先调用 capstone 汇编引擎，解析每一条指令读写的寄存器列表。执行前，判断指令指令读取的寄存器是否为污点；执行后，判断指令写入的寄存器是否被污点。

### trace 模拟执行

在污点执行过程中，可能有将污点结果与没有污点分析结果进行对比的需求，因此需要实现不带污点的执行。对于不带污点的执行，只需要不设置污点即可，封住 `trace_step`​ 函数，该函数传入的 state 不能设置污点。

```python
def trace_step(proj, state, run_until_pc=None, num_inst=None):  
    successors, _ = taint_step(proj, state, run_until_pc, num_inst, check_every_inst=False)  
    return successors
```

### **敏感操作检测**

污点分析的最后一个环节是**敏感操作检测(sink)。** 在反混淆场景中，sink 需要根据具体的需求来制定。以 bcf 混淆为例，sink 就是 state 发生分叉，即遇到了污点传播到的条件跳转指令。

污点检查函数 `taint_check_expr`​ 在前面已经讨论过。

### deflat

通过 deflat 符号执行脚本去混淆，主要去除控制流平坦化和虚假的控制流。

- deflat 脚本下载

>下载：https://github.com/cq674350529/deflat.git

### D810

- D810 环境搭建

D810下载：[eshard / D810 · GitLab](https://gitlab.com/eshard/d810)

直接下载插件，然后将 d810 目录和`D810.py`文件复制到 ida 中的`plugins`文件夹下。

使用插件需要安装依赖，进入`IDA\python311\Scripts`目录，然后执行以下命令：

```shell
./pip.exe install PyQt5
./pip.exe install z3-solver
```

快捷键`ctrl+shift+D`可以调出 d810 配置面板

## 控制流平坦化

- D810 去除

打开 D810 后，选择第二个规则。

![](assets/Pasted%20image%2020250609104733.png)

然后点击 start，等到右边出现绿色的loaded。

## 虚假控制流

Hikari BCF 虚假控制流，先不讨论它的简单去除方法，以这个为例子来验证用于反混淆的污点分析框架特别合适。

一些方法

1. 简单方法：IDA Pro 把 data 段设置只读，借助编译优化自动优化（不讨论，确实可以，没有泛化能力）
2. unicorn 模拟执行（不能肯定回答某个条件跳转指令是否与 BCF 混淆相关，容易误报！）
3. 污点分析


模拟执行无法回答某个条件跳转指令是否与 BCF 混淆相关，但是可以回答条件跳转指令实际跳转的目标地址；污点分析框架无法回答被污点感染的条件跳转指令的运行时目标。把两者结合一下，就能做到准确识别 BCF 中的混淆条件跳转指令，并且分析运行时实际地址。模拟执行的过程仍然可以用 angr 来实现，只需要不添加污点源就是模拟执行！

目标：识别 BCF 引入的部分垃圾代码，如下图红色部分，特别是最后一条指令 `B.LS`​ 条件跳转，对于最后一个 BLS 条件跳转，还要分析它的目标，我们要修复它！
‍
![](assets/image-20250130162636-np871kw.png)


具体流程如下：

### 查找所有 BCF 位置

查找所有含有 BCF 的基本块，思路是通过 BCF 引入的全局变量交叉引用分析。

```python
def debcf_hikari_find_all(hikari_bcf_start, hikari_bcf_end):
    """
    find all hikari bcf in (hikari_bcf_start, hikari_bcf_end)
    only works in ida
    return a list of hikari bcf start address
    """
    block_list = []
    for start_ea in range(hikari_bcf_start, hikari_bcf_end, 4):
        xrefs = [ref.frm for ref in idautils.XrefsTo(start_ea)]
        block = ida_get_bb(xrefs[0])
        if block.start_ea not in block_list:
            block_list.append(block.start_ea)
    return block_list


def ida_get_bb(ea):
    f_blocks = idaapi.FlowChart(idaapi.get_func(ea), flags=idaapi.FC_PREDS)
    for block in f_blocks:
        if block.start_ea <= ea and ea < block.end_ea:
            return block
    return None
```

`debcf_hikari_find_all`​ 函数在 IDA 里面调用，返回所有含有 BCF 混淆的基本块地址列表。

### 分析流程

‍以基本块为单位，构建一个分析函数 `debcf_hikari(proj, addr, taint_area)`​，addr 是基本块地址，`taint_area`​ 是 BCF 的全局变量区间。该函数的目标是分析一个基本块，找出所有 BCF 相关的指令，特别是 BCF 相关的条件跳转，并分析条件跳转的目标地址，以便于后续的修补工作。

```
# addr = bcf 基本块开始地址  
init_state = proj.factory.blank_state(addr=addr)  
init_state.options.add(angr.options.CALLLESS) # 忽略函数调用  
init_state.options.add(angr.options.LAZY_SOLVES) # 延迟求解  
init_state.options.add(angr.options.ZERO_FILL_UNCONSTRAINED_MEMORY) # 未初始化内存设置 0  
init_state.options.add(angr.options.ZERO_FILL_UNCONSTRAINED_REGISTERS)# 未初始化寄存器设置 0
```

‍污点分析、模拟执行混合

```
trace_state = init_state.copy() # 用于追踪，trace 实际运行值  
taint_state = init_state.copy() # 用于污点分析，查找 BCF 相关指令
```

‍taint 污点源标记，标记 BCF 全局变量区间

```
taint_memory(taint_state, taint_area[0], taint_area[1] - taint_area[0])
```

‍分析过程

污点分析，模拟执行同步进行，每次循环分析一条指令。

首先污点分析执行一条指令，然后模拟执行执行一条指令，接下来做 sink 检查

- 如果污点分析当前指令被污点感染，则将当前指令地址视为垃圾指令

- 如果污点分析当前指令被污点感染且是条件跳转指令，从 trace 中取出目标跳转地址

整个分析过程的循环直到遇到跳转指令结束（或者 csel）。‍

对于每一条指令，下面是代码过程

污点分析执行一条指令

```
# 执行指令（taint）  
taint_successors, tainted_list = taint_step(proj, taint_state, check_every_inst=True, num_inst=1)
```

模拟执行执行一条指令

```
# 执行指令（trace）  
trace_successors = trace_step(proj, trace_state, num_inst=1)
```

sink 检查

```
if len(tainted_list) == 1 and len(trace_successors) >= 1 and len(taint_successors) >= 1:  
    junk_code_list.append(tainted_list[0])  
    if insn.mnemonic.startswith('b.'):  
        patch_list.append((insn.address, 'B', trace_successors[0].addr))  
        break # 结束分析
```

最后切换下一条指令执行

```
trace_state = trace_successors[0]  
taint_state = taint_successors[0]
```

‍

完整实现

```
def debcf_hikari(proj, addr, taint_area):  
    """  
    addr: the address of the block to analyize  
    taint_area: a tuple (start_addr, end_addr)  
    """  
    patch_list = []  
    junk_code_list = []  
    init_state = proj.factory.blank_state(addr=addr)  
    init_state.options.add(angr.options.CALLLESS)  
    init_state.options.add(angr.options.LAZY_SOLVES)  
    init_state.options.add(angr.options.ZERO_FILL_UNCONSTRAINED_MEMORY)  
    init_state.options.add(angr.options.ZERO_FILL_UNCONSTRAINED_REGISTERS)  
    
    trace_state = init_state.copy()  
    taint_state = init_state.copy()  
    
    taint_memory(taint_state, taint_area[0], taint_area[1] - taint_area[0])  
    
    while True:    
        insn = trace_state.block().capstone.insns[0]  
        regs_read_ids, regs_write_ids = insn.regs_access()  
      
        reg_write_list = [insn.reg_name(r) for r in regs_write_ids]  
        reg_read_list = [insn.reg_name(r) for r in regs_read_ids]  
      
        if 'nzcv' in reg_write_list:  
            reg_write_list.remove('nzcv')  
      
        logger.info("debcf_hikari: pc: %x %s %s" % (trace_state.addr, insn.mnemonic, insn.op_str))  
    
        # 执行指令（taint）  
        taint_successors, tainted_list = taint_step(proj, taint_state, check_every_inst=True, num_inst=1)  
      
        # 处理 taint csel 指令  
        if len(tainted_list) == 1 and insn.mnemonic == 'csel':  
            setattr(trace_state.regs, reg_read_list[1], 1)  
            setattr(trace_state.regs, reg_read_list[2], 2)  
      
        # 执行指令（trace）  
        trace_successors = trace_step(proj, trace_state, num_inst=1)  
      
        if len(tainted_list) == 1 and len(trace_successors) >= 1 and len(taint_successors) >= 1: # 如果 taint 成功，则说明是 junk code  
            logger.info("debcf_hikari: junk code pc: %x" % tainted_list[0])  
            junk_code_list.append(tainted_list[0])  
          
            if insn.mnemonic == 'csel':  
                operands = insn.operands  
                dst_reg  = insn.reg_name(operands[0].value.reg)  
                src1_reg = insn.reg_name(operands[1].value.reg)  
                src2_reg = insn.reg_name(operands[2].value.reg)  
                # trace_state 是指令执行前的状态，trace_successors[0] 是指令执行后的状态  
                dst_val = trace_successors[0].regs.get(dst_reg)  
                src1_val = trace_state.regs.get(src1_reg)  
                src2_val = trace_state.regs.get(src2_reg)  
                dst_val =  trace_successors[0].solver.eval(dst_val)  
                src1_val = trace_state.solver.eval(src1_val)  
                src2_val = trace_state.solver.eval(src2_val)  
                if dst_val == src1_val:  
                    patch_list.append((insn.address, 'MOV-REG-REG', dst_reg, src1_reg))  
                elif dst_val == src2_val:  
                    patch_list.append((insn.address, 'MOV-REG-REG', dst_reg, src2_reg))  
                else:  
                    logger.error("debcf_hikari: unknown csel pc: %x" % insn.address)  
                break  
          
            if insn.mnemonic.startswith('b.'):  
                if len(trace_successors) != 1:  
                    logger.warning("debcf_hikari: more than one successor in %x" % insn.address)  
                else:  
                    patch_list.append((insn.address, 'B', trace_successors[0].addr))  
                logger.info("debcf_hikari: find cond branch pc: %x" % insn.address)  
                break  
          
        if insn.mnemonic in ['ret', 'b']:  
            logger.info("debcf_hikari: find branch pc: %x" % insn.address)  
            break  
      
        if len(trace_successors) != 1:  
            break  
        else:  
            trace_state = trace_successors[0]  
            taint_state = taint_successors[0]  
    
    return patch_list, junk_code_list
```

这段代码能够完美处理 hikari 编译的 aarch64 cff+bcf，即使有控制流平坦化 cff 也能正常工作，且 patch 完成后不影响程序的正常逻辑，在 miniz 压缩算法库完成测试。

### D810 去除

打开 D810 后，同样选择第二个规则。

![](assets/Pasted%20image%2020250609104733.png)

然后点击 start，等到右边出现绿色的loaded。


## 指令替换

指令替换去除可以通过 D810 去除，但是也不能完全去除，效果不是很好。

## 字符串加密

## 参考