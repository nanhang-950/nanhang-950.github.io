---
title: "论文笔记：Fuzzing: A Survey for Roadmap"
date: 2025-11-30 23:30:38
categories: Paper
tags: 
mathjax:
top:
---

## 0x00 前言

看了 arttnba3 师傅的关于这篇论文的博客想要好好读一下这篇论文，但是没有找到免费的 PDF 文档，而 ACM 官网下载要钱，而且还是doler。所以就直接看 arttnba3 师傅的博客了，这篇文章就是看 arttnba3 师傅的博客记的笔记，几乎就是完全转载了 arttnba3 的博客，仅对格式做了一些小小的修改，特此声明一下。

>原文链接：[【PAPER.0x00】论文笔记：Fuzzing: A Survey for Roadmap - arttnba3's blog](https://arttnba3.cn/2022/10/30/PAPER-0X00-FUZZING_A_SURVEY_FOR_ROADMAP/#4-5-Fragment-Recombination)

摘要：**Fuzz testing**（fuzzing，即模糊测试）在检测安全漏洞中大放异彩，其通过生成大量的测试用例（test cases）并观测执行结果来寻找漏洞，并且已在大量的应用中发现了上千个漏洞。虽然非常高效，fuzz 仍缺乏系统化的对其缺陷的分析。

- fuzz 需要缩小**输入空间**（input space）与**缺陷空间**（defect space，触发缺陷的输入）间的差距；在一个应用当中，漏洞（defects）的存在是分散的（spare），这意味着 defects space 要比 input space 小得多。
- fuzzing 生成大量的测试用例进行重复测试——这需要一种自动化的方法；由于查询与漏洞的复杂性，自动化地执行不同的程序会是一个挑战。

## 0x01 简介

软件漏洞是计算机系统中的一个严重的问题，而 **Fuzz testing** （模糊测试）已经成为最成功的检测程序漏洞的方法之一，其通过生成大量的测试用例来重复测试目标程序并观察其**异常**（exception）——安全漏洞的标志（indicator）

Fuzzing 通常有着一组**种子**（seeds）：interesting inputs，新的输入的生成则基于这组种子进行无限的变异（mutate）。

虽然 fuzzing 在发现安全漏洞上获得了巨大的成功，在开发高效的漏洞检测方案上仍存在着缺陷，如 Fig.1 所示，三个主要的缺陷是：输入中分散的漏洞空间，严格的有效输入空间，多目标的自动化执行。
<img src="/assets/Pasted%20image%2020250621145238.png" style="zoom: 80%;" >

- **Gap 1: 输入中分散的漏洞空间**：在应用程序中的漏洞分布是分散的，而仅有部分特定的输入能够触发漏洞；浅显的漏洞可以在短时间内被 fuzz 到，但许多安全漏洞需要测试复杂的执行路径并解决严格的路径约束，因此一个高效的 fuzzing 算法需要同时对 _待测试程序_ （program under test， **PUTs**）与 _安全缺陷_ （security flaws）足够精通，以在一个更有可能存在漏洞的代码区域驱动计算资源

- **Gap 2: 严格的有效输入空间**：大部分程序有着自己的输入空间，而现代程序都相当复杂，需要更复杂的特化输入空间，因此如何生成有效输入同样是个挑战；此外，为了提高 fuzzing 的效率，生成的输入应当使用不同的执行状态（例如 _代码覆盖率_ ），这需要更先进的方案来生成有效输入；若缺乏对 PUTs 的系统化分析，几乎不可能精确地限制输入空间（例如 PDF 文件的变异生成可能会违反 PDF 规范）

- **Gap 3: 多目标的自动化执行**：由于 fuzzing 大量重复地测试 PUTs，这需要高效的自动化方法。PUTs 与漏洞都是多种多样的，有的程序可以简单直接地被自动化地 fuzz（例如命令行程序），但许多程序在自动化测试前都需要做大量的工作（例如硬件）；此外，安全缺陷同样需要自动化的 indicator 以记录潜在的真正漏洞，**程序崩溃**是一个常用的 indicator 因为其可以被 OS 自动捕获，但有的安全缺陷**并不会表现出崩溃**（例如条件竞争），这需要精心设计的 indicator

业界在缩小这些缺陷上做出了许多努力。在本篇论文中，研究者系统化地回顾与分析了 fuzzing 的缺陷与解决方案，同时考虑了广度与深度。


## 0x02 Fuzzing 概述

<img src="/assets/Pasted%20image%2020250628152009.png" style="zoom: 80%;" />

首先介绍一些术语（Terminologies），如 Fig2 所示：

- **seed**（种子输入）：被保留的能完成更好的 fitness 的输入（例如提供新的覆盖率）；
- **fitness**（适应度）：对一个 input/seed 的质量的测量；
- **power schedule**：决定了分配给 seeds 的 energy；
- **energy**：分配给当前 fuzzing round 的变异数量；
- **fuzzer**：fuzzing 算法的实现，一般指 Fuzzing 工具。

如 Fig2 所示，fuzzing 由三部分组成：

- **input generator**（输入生成）：负责向 executor 提供输入
- **executor**（执行）：负责执行输入
- **defect monitor**（监控）：负责检查是否发现了新的执行状态或缺陷（例如crashes）

基于输入的生成方式，fuzzing 可以分为：

- **基于生成的**（generation-based）：基于 _文法_ （grammars）或 _有效语料库_ （valid corpus）从头开始生成；如 Fig2 所示，其从一组种子中直接获得输入
- **基于变异的**（mutation-based）：对现有的种子进行 _变异_ （mutate）以获得新的输入；对给定的一组种子，基于变异的模糊测试通过 seed schedule、byte schedule、mutation schedule 以获得输入。

>需要注意的是，fuzzing 并不需要经历 Fig2 中的所有步骤，例如基于生成的模糊测试并不执行 byte schedule 或 mutation schedule，但关注于从初始输入文件中选择最优的种子组。

基于执行时观测到的信息量，fuzzing 可以分为：

- **黑盒**（blackbox）：黑盒模糊测试并不知道每次执行的内部状态，通过使用输入格式化或不同的输入状态来进行优化。
- **白盒**（whitebox）：白盒模糊测试对每次执行的内部状态是全部得知的，这使其能系统化地探索目标程序的状态空间；其通常使用 concolic execution（例如 dynamic symbolic execution，即动态符号执行）来分析目标程序。
- **灰盒**（greybox）：灰盒模糊测试获得的执行状态信息在黑盒与白盒之间，例如许多 fuzzer 都使用**边界覆盖率**（edge coverage）作为内部执行状态。

最通用的执行状态便是**代码覆盖率**（code coverage，例如 CFGs（control flow grphs）中的基本块（basic block、边（edges））），覆盖率的基本假设用法是：发现更多的执行状态（例如新的覆盖率）能提高发现漏洞的概率。因此覆盖率指导（coverage-guided）的模糊测试的目标便是覆盖更多的代码。

>但执行状态并不限制于代码覆盖率，对面向对象程序而言也可以是执行的合法性，对协议实现开源是状态机，对并发实现开源是 alias coverage，对深度学习模型开源是神经覆盖率。

Fuzzer 通常使用 崩溃（crashes）作为安全漏洞的指示器，因为 crashes 提供了直接的自动记录（OS 回自动发出信号告知程序崩溃），然而有的缺陷并不会显示出 crashes，因此 fuzzer 使用其它的指示器，例如 physical safety violation。

但 indicators 仅显示了可能的安全问题，还需要安全工具或人工确认这是一个 _漏洞_ （vulnerability）。


## 0x03 Fuzzing 理论

为了提高发现漏洞的概率，fuzzer 在执行过程中使用反馈（feedback）机制，例如以执行状态或结果作为 fitnes，一个典型的 fitness 便是基于代码覆盖率（例如基本块或边）进行输入生成，但仅有代码覆盖率**并非一直都是可靠的**，就算可靠也可能收益不高（例如指数型数量的输入生成可能只带来线性的漏洞发现），因此一种常见的改进方法是优化模糊测试的过程或是为 fitness 丰富信息，Table 1 展示了不同的 fuzzer 的优化方法：

<img src="/assets/Pasted%20image%2020250628153048.png" style="zoom: 80%;" />

### 3.1 种子集选择

对种子集的优化关注于**最小化种子集的大小**，例如选择能覆盖所有已发现代码覆盖的一组最少的种子，因为过于富集的种子会在检验已探测代码区域上浪费计算资源。

### 3.2 种子调度

**种子调度**（Seed Schedule）期望解决如下问题：

- 在下一轮中选择哪个种子。
- 为该种子分配的**时间预算**（time budget）；大部分 fuzzer 实际上选择优化对被选取种子的变异次数。

由于 PUTs 与漏洞的复杂性，未发现代码覆盖率与未发现漏洞是不可知的，我们无法知道一个输入是否能触发漏洞，类似地在检索代码之前我们也不能获得程序行为的概率分布，因此数学上我们几乎不可能找到一个全面的优化解法，因此研究人员基于多种优化方法来近似地解决这个问题。

#### 3.2.1 Fitness by Bugs

通常而言在模糊测试过程中使用两种 fitness 进行优化：

1. 基于漏洞；
2. 基于执行状态（例如代码覆盖率）。

由于 fuzzing 的目的是发现漏洞，发现漏洞的数量便是一种最简单的 fithess，一种方法便是在随机/顺序选择种子的时候调度每个种子的时间预算，在不考虑执行状态的情况下，最大化漏洞数量问题可以被简化为一个整数线性规划（Integer Linear Programming，ILP）问题，即在线性约束下最大化漏洞数量——以解决这样的 ILP 问题来自动计算每个种子的时间预算。

另外一种认知是将漏洞发现的过程视作**带权奖券收集问题**（Weighted Coupon Collector’s Problem）：fuzzing 中发现的每个独特的漏洞都被视作一种“奖券”，WCCP 期望以此预测发现下个“奖券”所需要的尝试的数量（时间预算）。

ILP 与 WCCP 都是为了将更多的时间于是分配给更有潜力的种子以发现更多漏洞。

#### 3.2.2 Fitness by State Transition（Markov Chain）

由于漏洞在 PUTs 中的分布是分散的，若以已发现漏洞为 fitness，则 fuzzing 只会关注与已发现漏洞相关的代码区域，这有可能无法获得更多的代码覆盖，这种情况下需要复杂条件的深层漏洞则并不会被 fuzzing 出来。

为了缓解这个问题，fuzzer 基于**执行状态**（execution state，例如代码覆盖率）计算 fitness，因为执行状态能提供更多的信息；现有的 fuzzer 通常使用代码覆盖率来计算 fitness，因为更高的代码覆盖率意味着更高的发现漏洞的可能性。

如 Fig2 所示，若模糊测试能成功地表示状态迁移，其能高效地指引模糊测试以探索未发现的状态，一个热门的建模方法便是**马尔科夫链**（Markov Chain），其中一种解决方案是将 CFGs 中的一个基本块视作一个状态，状态转移即为基本块的执行迁移，fuzzer 通过在 fuzzing 过程中记录基本块的跳转频率来计算概率。

>简而言之，马尔科夫链维护一个概率表（probability table），元素 _pij_ 表示从状态 _i_ 转移到状态 _j_ 的概率
>![](/assets/Pasted%20image%2020250630225028.png)

有了基于马尔科夫链计算的 fitness，fuzzer 可以指导算力的分配，或是选择使用符号执行解决最困难的路径（最低的转移概率）；通常而言，转移概率越低，fitness 越好，因为能轻易到达的代码区域会比难以到达的区域更容易被充分探索。

基于变异的模糊测试中通过对种子进行变异生成新的输入，每次输入运行了一条执行链，这提供了另一种基于马尔科夫链的视角：状态被定义为一个输入的一条执行路径，状态转移则为对输入 _ti_ 的变异，生成新的输入 _tj_ ，即由路径 _i_ 迁移到路径 _j_ ，fuzzer 在 fuzzing 过程中动态计算路径迁移的概率。

论文给出了这个例子：[AFLFast](https://ieeexplore.ieee.org/abstract/document/8233151)

#### 3.2.3 Fitness by State Transition（Multi-Armed Bandit）

马尔科夫链需要基于对所有状态已知来做出合适选择，但在 fuzzing 过程中并非所有状态都已被执行，因此马尔科夫链并非最优解。

对于基本块转移，可以在数据统计中使用 rule of three；对于路径转移，可以使用轮询调度算法将时间运算平均分配给每个种子，然而这种方法无法决定什么时候从轮询调度切换到马尔科夫链——在遍历所有种子与关注特点种子间进行平衡，这便是一个经典的`exploration vs. exploitation` 问题。

一个更好的解决 `exploration vs. exploitation` 问题的方法便是使用 **多臂老虎机**（Multi-Armed Bandit，MAB）表示路径转移：种子 _ti_ 被视作一条“臂”，“奖励”则是由种子 _ti_ 生成的一条新路径的发现。

论文给出了这个例子：[EcoFuzz](https://www.usenix.org/conference/usenixsecurity20/presentation/yue)

#### 3.2.4 Fitness by State Discovery

马尔科夫链和多臂老虎机都能表示程序的状态迁移，然而模糊测试的基本目标是发现新的状态（新的代码覆盖率、新的漏洞、新的崩溃），由此模糊测试过程也可以表示为一个 **物种发现问题**（species discovery problem）： _生态学家从野外收集大量样本，样本中的物种可能丰富或稀少，生态学家以此推断集聚的性质，包括未发现的样本_ ——类似地，**由 fuzzer 生成的输入即为“已收集到的样本”，输入空间即为“集聚”（assemblage）**，fuzzing 基于特定标准将输入进行分类。

> 例如一条执行路径可以是一个物种，执行这条路径的输入属于该物种，一个稀有物种即为较少输入执行到的路径，这引导着 fuzzer 去发现新的“物种”——新的路径→新的状态

论文给出了这个例子：[Entropic](https://mboehme.github.io/paper/FSE20.Entropy.pdf)

### 3.3 字节调度

**字节调度**（Byte Schedule）决定了 _选择种子中一个字节来变异_ 的频率。大部分 fuzzer 基于执行信息来试探性地或随机地选择字节，这需要比 seed schedule 对程序行为有着更深刻的了解（例如路径约束或数据流），由此 fuzzer 可以关注于一个不那么复杂的问题——字节如何影响模糊测试的过程，称为字节的**重要性**（importance）

由于大部分灰盒 fuzzer 使用 _边覆盖率_ 来测试 PUTs，第一种方法便是将重要性定义为 _字节如何影响分支行为_

论文给出了这些例子：[NEUZZ](https://arxiv.org/abs/1807.05620)、[MTFuzz](https://arxiv.org/abs/2005.12392)

另一种量化字节的重要性的方法是基于种子的 fitness 进行定义，在 fuzzing 过程中可以关注能提升 fitness 的字节：若对于一个字节的改变提升了种子的 fitness，则增加该种子的分数

论文给出了这个例子：[AFLChurn](https://dl.acm.org/doi/10.1145/3460120.3484596)

### 3.4 Mutation Operator Schedule

如 Fig2 所示，输入生成器的最后一步是选择一个变异器（mutation operator，即mutator）来对选择的字节进行变异，**变异调度**（mutation schedule）决定了下次变异所用的变异器。

论文给出了这些例子：

- [Classfuzz](https://dl.acm.org/doi/10.1145/2908080.2908095)：使用 _马尔科夫链蒙特卡洛方法_ （Markov chain Monte Carlo） 建模变异调度。
- [MOPT](https://arttnba3.cn/2022/10/30/PAPER-0X00-FUZZING_A_SURVEY_FOR_ROADMAP/)：使用 _粒子群优化_ （Particle swarm optimization）建模变异选择过程。

### 3.5 Diverse Information For Fitness

itness 除了调度种子、字节、变异器以外，还可以被用于指导种子存留：输入由种子变异而来，若一个输入探索到新的执行状态，其便被保留为新的种子，在种子调度中常选择新的种子；大部分的覆盖率指导的 fuzzer 基于边覆盖率来保留种子

为了提高发现漏洞的能力，需要更敏感的代码覆盖率带来更多的执行状态信息；另一方面，新类型的 fitness 也被为一些特殊场景设计了出来，例如深度学习模型或机器装置。

<img src="/assets/Pasted%20image%2020250630230235.png" style="zoom: 80%;" />

#### 3.5.1 敏感代码覆盖率

fitness 的敏感度标识区分执行状态的能力，大部分覆盖率指导的 fuzzer 使用一个位图来提供边覆盖率信息：

- 位图中的每个元素下标表示一个**边标识符**（edge identifier），并为边标识符计算哈希值 _hash(bi, bj)_ ，其中 _bi_ 与 _bj_ 为随机指派的基本块标识符（block identifier）

尽管这种方法执行得很快，但其舍弃了对边覆盖率的预测，同时这种维护边覆盖率的实现导致了**边碰撞**（edge collision）问题（例如两条不同的边被分配了同一个标识符），为了分配唯一的边标识符，fuzzer 需要小心地分配块标识符且提供更精确的哈希函数

> 如 Fig3.a 所示，若 _idAB_ == _idAC_ 且 _idBD_ == _idCD_ ，则路径`ABD`与`ACD`被当作同一条路径，在这种设想下 fuzzer 无法获得新的覆盖率，其会忽略漏洞路径`ACDEG`。

fuzzer 通过位图便能确定一个输入是否产生了新的边；特别地，fuzzer 维护一个总体位图（overall bitmap，独立执行位图（bitmap of individual execution）的集合），在确定新的边时，fuzzer 将独立位图与总体位图对比，以检查这条新的边是否存在于独立位图中，然而位图集合丧失了执行信息

> 例如若 Fig3.a 中的路径`ABDEG`与`ACDFG`已经被训练，则新的边`ACDEG`将不会被作为新的种子保留，因为在总体位图中早已存在所有的边。

一个解决方案是研究独立位图的融合，但由于融合位图会带来太多的种子，这需要在 fuzzing 效率与敏感覆盖率间平衡，一个潜在的解决方案是使用**动态主成分分析**（dynamic principle component analysis）来减少数据集的维度，其他的解决方案则是为边覆盖率提供额外信息，包括：**边哈希**（edge hash）、**调用上下文**（calling context）、**多级覆盖率**（multilevel coverage）、**代码复杂度**（code complexity）。

对位图的改进关注于搜寻更多的代码覆盖，然而这样的改进可能没法探索复杂的执行状态，因此一个有效的解决方案便是通过**人为监督**（human-in-the-loop）来指导对复杂执行状态的探索

> 例如 Fig3.b 为一段迷宫代码，`(a,b)` 代表在迷宫中的位置，为了触发`bug()`，`(a,b)`应当有着特定的值，然而`switch`仅有四条边，可以被快速遍历，在这之后 fuzzing 便失去了对到达漏洞位置的指引；此时分析者可以让模糊测试过程探索`(a,b)`的不同值。

此外，比较条件的值比二进制结果更加敏感，即`examined`或`not examined`。

> 例如 Fig3.a 中若`(x[3] - 0x44)`的值是可知的，则 fuzzer 便能选择更能满足条件`if (x[3] == 0x44)`的种子。

#### 3.5.2 Diverse Fitness

由于模糊测试可用来检测多种应有的缺陷，而代码覆盖对于模糊测试而言并非一直都是最好的反馈，因此多种类型的 fitness 被针对特定的应用程序或缺陷设计了出来：

- _Legality of execution result_ ：一门 OOP 语言（比如 Java）由一个方法调用序列组成，非法的执行结果会抛出异常；在模糊测试过程中会生成并维护能探索更多新的及合法的目标状态的新的调用序列；
  
- _State Machine of protocol implementations_ ：由于协议的复杂性，fuzzer 通常通过搭积木的方式来推断出状态机：以一组种子起始变异以获取新的状态，基于状态机分析漏洞点并搜寻可能存在漏洞的状态转移；
  
- _Safety policy of robotic vehicles_ ：机器装置的物理/功能安全需要 _安全策略_ （safety policy，例如机器的温度限制），因此可以保留接近违反安全策略的输入作为种子进行变异；
  
- _Fitness for deep learning system_ ：对**深度学习系统**（Deep Learning Systems，DLSs）的模糊测试设计了几种不同的 fitness。
  

> 例如神经元覆盖率以发现**极端场景**（corner cases），损失函数以增强训练数据，**操作符层**（operator-level）覆盖率以探索深度学习**推理机**（inference engines）

- _Validation log for Android SmartTVs_ ： validation log 可以被用来推断合法的输入以及获取输入边界，这为 fuzzer 提供了高效的种子且缩小了输入空间；
  
- _Behavioral asymmetry of testing_ ：在**差异测试**（differential testing）下，可以通过在相同功能实现的相同输入上观测到不同行为来发现漏洞；
  
- _Alias coverage for data race_ ： alias coverage 通过跟踪一对可能交互的内存访问，以此发现由于缺乏合适的同步机制导致的条件竞争漏洞；

- _Dangerous locations for bugs_ ：危险区域是容易触发漏洞的区域，fuzzer 可以直接将资源集中在上边进行模糊测试以提高效率。

> 例如对并发漏洞而言可以是会造成对原子性的违反、条件竞争等的代码区域，对于非并发漏洞而言可以通过补丁测试、崩溃复现、静态分析报告、信息流检测来获得，此外危险区域也可以是内存访问、sanitizer 检查或是 commit 记录

### 3.6 Evaluation Theory

一个合适的**评估**（evaluation）可以在模糊测试过程中帮助其提高表现，包括有效实验语料库、公平的评估环境、合理的模糊测试时间、全面的对比指标。

>**Gap 1**：模糊测试理论缩小了输入空间与缺陷空间的差距，基于程序表现（如漏洞到达、状态转移、状态发现）规划模糊测试过程，大部分的理论规划了种子调度，几乎所有的 fuzzer 基于遗传算法规划种子存留。


## 0x04 搜索输入空间

为了缩小输入空间、提升模糊测试的性能，fuzzers 将一个输入中的**关联字节**（related bytes，例如组成同一数据结构、影响同一路径约束、符合同一文法）分组并为每一组使用特定的变异器（包括字节变异与块变异）。

> 假设输入空间中有`a* b`字节，将其等分为 a 块，则相较于 _256a*b_ 而言，对于特定路径约束的搜索空间仅为 _a * 256b_

在路径约束求解中，对关联字节的关注同样能缩小搜索空间，例如 Fig4.a 在第 13 行的约束满足的情况下第14行仅与一个字节相关联。

一种特殊的输入是为如协议、编译器等进行高度结构化的输入，如 Fig4.b 中代码需要一个特殊起始格式的输入。

<img src="/assets/Pasted%20image%2020250630230345.png" style="zoom: 80%;" />

Table 2 中的 fuzzers 的主要贡献是缩小输入空间：

<img src="/assets/Pasted%20image%2020250630230401.png" style="zoom: 80%;" />

### 4.1 Byte-constraint Relation

大部分的路径约束仅被一小部分输入所影响，因此若 fuzzer 仅变异关联字节，则能通过缩小输入的搜索空间从而显著地提升性能。

> 例如我们需要变异一个字节数组 a[10] 需要生成 _25611_ 个输入，但若是我们知道仅有 a[2] 的值是有用的，则我们仅需要生成 256 个输入。

在获得字节约束关系后，可以随机地或从0~255 进行变异，但都比较低效；若在字节关系的推断过程中可以获得比较指令的值，fuzzer 可以在变异时选择通过路径约束的值。

此外，模糊测试可以使用 _坡度下降算法_ （gradient descent algorithm）来变异关联字节并逐渐解决路径约束。

#### 4.1.1 Dynamic Taint Analysis

**动态污点分析**（Dynamic Taint Analysis，DTA）是在构建输入与路径约束的关系中常用的一项技术，其通过**在输入中进行标记后在运行中传播标签（label）并检查获取到标签的变量**的方式来构建变量与数据的关联。

fuzzer 可以使用 DTA 来构建输入与**安全敏感点**（security-sensitive points，例如条件跳转或系统调用）间的关系。

#### 4.1.1 Relation Inference

DTA 需要大量的人工且可能获得不准确的关系（due to implicit data flow）。由于 fuzzing 过程中需要大量执行测试用例，一种轻量的解决方案是在运行时推断字节关联，有两种具体方案：

- 观测是否对一个字节的变异改变了变量的值，这意味着该字节可能与变量、比较指令或分支相关联
- 基于深度学习构建输入字节与分支行为间大概的关系

### 4.2 Concolic Execution

**混合执行**（Concolic Execution，aka dynamic symbolic execution）将程序变量视作**符号变量**（symbolic variables），跟踪路径约束并使用约束求解器来为特定路径生成具体输入：通过求解路径约束来缩小输入空间。

<img src="/assets/Pasted%20image%2020250630230525.png" style="zoom: 80%;" />

同时使用符号执行与模糊测试的技术称之为**混合模糊测试**（hybrid fuzzing）或**白盒模糊测试**（whitebox fuzzing）：使用模糊测试来执行目标程序中的执行路径＋使用符号执行来求解执行路径中的约束。

由于为每条执行路径都使用符号执行会非常耗时，因此当 fuzzing 无法获得更多状态时，混合执行被用以解决 fuzzing 无法满足的路径约束。

混合模糊测试的一个改进便是为混合执行排序出最难的路径供其解决，也可以通过开发一个大概的约束求解器来提升，通常**可满足性模理论**（satisfiability modulo theory，STM）求解器（例如 z3 或 MathSAT5）被用来求解路径约束，但存在复杂约束及路径爆炸的问题，为了缓解这个问题，约束求解器仅符号化被输入影响的路径。

另一个改进便是让约束求解器使用灰盒模式，例如，其使用线性函数以接近约束行为，因为大部分的路径约束被发现都是趋向于是线性的或单调的。

研究人员也开始使用模糊测试来求解路径约束，例如 [JFS](https://dl.acm.org/doi/10.1145/3338906.3338921) 将 SMT 公式翻译为程序并使用覆盖率指导的模糊测试来探索程序，模糊测试生成的输入到达特定区域或相应的程序时意味着解出了 SMT 公式。

约束求解器也可以基于目标的特性进行改进，[Pangolin](https://ieeexplore.ieee.org/document/9152662) 使用多面路径抽象（polyhedral path abstraction）来解决嵌套路径约束（nested path constraints），这种方法会保留历史约束的解答空间（solution space）并重用解答空间以满足当前路径约束的可达性。

> 例如对于 Fig4.a 中的第14行的约束，输入首先要满足第13行的约束

为了在需要高度结构化的输入的程序中使用混合模糊测试，[Godefroid](https://dl.acm.org/doi/10.1145/1375581.1375607) 将文法中的词素（token）符号化为符号变量，并使用上下文无关的约束求解器（context-free constraint solver）来生成新的输入。

### 4.3 程序转换

对模糊测试而言，程序转换（Program Transformation）的目的是移除防止模糊测试发现更多执行状态的完整性检查，通过移除这些检查，模糊测试可以探索到目标程序更深处的代码并暴露出潜在的漏洞，但这也会引入一些误报（false positives），可以通过符合执行进行验证。

因此 Program Transformation 通过聚焦于可能触发漏洞的输入来缩小探索空间。

### 4.4 输入模型

许多应用程序都需要高度结构化的输入，例如协议实现、系统调用等，**输入模型**（input model）指定了构造高度结构化的输入的规则，包括结构体、格式、输入的数据约束，即违反语法或语义的输入会在一开始就被拒绝，由此**输入空间便被限制于输入模型**。

#### 4.4.1 Accessible Models or Tools

基于空白规范（bare specifications）的输入生成需要繁重的工程工作，复杂规范的解析也非常容易出错，因此研究社区为一些高度结构化的输入开源了一些工具

> 例如 [QuickCheck](https://dl.acm.org/doi/10.1145/357766.351266) 和 [ANTLR](https://www.antlr.org/)，例如 [NAUTILUS](https://www.ndss-symposium.org/ndss-paper/nautilus-fishing-for-deep-bugs-with-grammars/) 与 [Superion](https://www.researchgate.net/publication/335427315_Superion_Grammar-Aware_Greybox_Fuzzing) 便基于 ANTLR 生成输入。

在一些场景下输入模型也可以是输入的类型（例如 API 参数或物理信号）。

#### 4.4.2 Integration of Implementations

另一个前景较好的方案是将模糊测试与目标应用进行集成，这样的集成允许模糊测试通过客制化输入生成过程来测试预期性能。

> 例如 [TLS-Attacker](https://dl.acm.org/doi/10.1145/2976749.2978411) 创造了能基于每个段的类型变异输入的框架，并能改变协议消息的顺序

#### 4.4.3 Intermediate Representation

另一个复杂的方案是将输入模型转化为一种**中间表示**（Intermediate Representation）：将原始输入文件翻译为更简单且更统一的 IR，fuzzer 基于 IR 进行变异后再翻译回原始输入格式。

这种变异策略能在保持了语法与语义的正确性的同时生成了多种输入。

### 4.5 Fragment Recombination

另一种生成输入的方式是通过**碎片重整合**（fragment recombination）：将输入文件分成许多的小块（fragments），通过从不同文件中整合碎片来生成新的输入，同时每个碎片应符合规范以确保语法正确性。

<img src="/assets/Pasted%20image%2020250630231005.png" style="zoom: 80%;" />

如 Fig.5 所示，fuzzer 首先将输入文件解析成一棵保持语法正确性的树（例如 AST），这需要一个有效的输入语料库来解析输入，同时 fuzzer 还需要为语料库收集此前造成错误行为的有问题的输入。

在此前曾经发现漏洞的区域或附近仍有可能存在新的漏洞，而有问题的输入已执行了能造成错误行为的复杂路径，因此碎片重整合可能会执行同样或相似的路径，这有利于探索更深的代码。

在第二阶段输入被碎片化后会存放到碎片池中，由于输入被解析成 AST，fuzzer 可以使用**非终止节点**（non-terminals）来组成新的子树，在重整合碎片时基于 随机/遗传算法/机器学习 来选用**语法兼容**（syntactically compatible）的碎片，此外语义正确性也对模糊测试的效率有重要影响。

> 例如 [CodeAlchemist](https://www.ndss-symposium.org/ndss-paper/codealchemist-semantics-aware-code-generation-to-find-vulnerabilities-in-javascript-engines/) 使用汇编约束标记碎片，仅在碎片满足约束时才进行整合


### 4.6 Format Inference

若输入模型不可用，推断输入格式也是前景较好的解决方案，且一个输入模型仅能生成一种特定格式的输入，因此**格式推断**（format inference）比基于模型的方案更加灵活。

#### 4.6.1 Corpus-based

一种直接的方案是从有效输入语料库进行推断。由于缺乏输入模型，研究者建立了端到端（end-to-end）的机器学习模型作为替代，**循环神经网络**（recurrent neural network，RNN）这一模型更合适生成结构化输入，但这一替代方案有可能受到生成非法输入的影响，因此训练数据需要相应的改进

> 例如 [DeepFuzz](https://faculty.ist.psu.edu/wu/papers/DeepFuzz.pdf) 生成语法有效输入的比例仅为 82.63%

模糊测试也可以基于有效输入语料库合成一种上下文无关语法来生成高度结构化的输入。

#### 4.6.2 Coverage-based

基于语料库的解决方案需要对输入规则的综合覆盖，可能会不实际，此外其并没有使用内部执行状态的信息，这可能会造成较低的代码覆盖率。

输入的格式指示了输入中不同字节的关系，因此基于代码覆盖，fuzzer 可以推断字节到字节的（byte-to-byte）关系来启动模糊测试。

> 例如 [GRIMOIRE](https://www.usenix.org/conference/usenixsecurity19/presentation/blazytko) 使用代码覆盖来推断目标程序所需的输入格式。

#### 4.6.3 Encoding Function

与上述关注于输入的方法不同，有的 fuzzer 搜索会编码输入格式的代码区域，因为这类代码与生成结构良好的（well-structured）输入相关，故 fuzzer 在编码格式前进行变异。

尽管 PUTs 的源码可能没法获取，但他们所生成的结构良好的输入则不然，例如有的社区会开源一些生成高度结构化输入的工具。

对 IOT 设备而言，大部分都通过配套程序来控制，因此通过定位与编码格式相关的代码，变异可以在函数的参数或是计算格式的指令上完成。

> 例如 [IOTFuzzer](https://www.ndss-symposium.org/wp-content/uploads/2018/03/NDSS2018_01A-1_Chen_Slides.pdf) 便 hook 了这类函数并对其参数进行变异。

### 4.7 Dependency Inference

格式推断主要解决语法需求，这仍可能生成有着错误数据依赖项的输入，例如在 Fig.6 中的`snippet2`中，在`2-5`出现了一个由于`errf()`未定义导致的错误。

<img src="/assets/Pasted%20image%2020250630231123.png" style="zoom: 80%;" />

许多应用都需要在输入中有着正确的**数据依赖项**（data dependency），通常由一系列**语句**（statement）组成，包括系统调用、对象、APIs、ABIs等。

#### 4.7.1 Documents or Source Code

序列的数据依赖项通常通过静态分析来推断，因为许多应用都有相应的文档或源码，可以据此推断数据依赖项，并在模糊测试过程中在生成输入前先生成其**先决项**（prerequisites）。

但静态分析误报率高且会错过接口的依赖项，因此一个较好的解决方案是结合静态分析与动态分析。

#### 4.7.2 Real-world Programs

真实世界的许多程序通过命令行来调用接口，这便包含了数据依赖项，fuzzing 可以基于这些真实世界中程序的**切片程序**（program slicing）生成调用接口的新程序。

数据依赖项也可以通过分析**执行日志**（execution log）来推断，日志中明确包含接口的顺序信息（例如哪个接口先被执行），同时隐含了接口间的参数依赖项信息。

为了获得这些直接与间接的信息，模糊测试在执行过程中 hook 每个接口并记录自己所需的数据。

> **Gap 2**：输入空间的减小依赖于对语法/语义关联的输入字节的分组，分组的优点是在探索更多执行状态中提升效率，由此 fuzzing 更倾向于满足路径约束以探索被这些约束守护的更深的代码区域


## 0x05 自动执行

**自动执行**（Automatic execution）是模糊测试理论与输入空间减方法的基础，而成功的模糊测试需要：

- **自动重复地运行 PUTs**。大部分 fuzzer 都能测试命令行程序，但对于硬件或多语言而言不行。
- **对潜在漏洞的自动指示器**（automatic indicator）。当前 fuzzer 使用 crashes 作为潜在漏洞的标志，但如条件竞争一类的漏洞并不会触发 crash。
- **高速执行**。在相同的时间内检验更多的测试用例，以此增加发现漏洞的机会。

### 5.1 Automatic Execution of PUTs

对不同应用程序的自动化模糊测试需要不同的工程努力，本节介绍几种自动化模糊测试的方法。

#### 5.1.1 Command-line Programs

模糊测试在测试命令行程序时通过子进程运行 PUTs 并将所需**选项**（options）与输入喂给程序，同时在执行 PUT 时其并不会重复所有的步骤，而是克隆出子进程以略过预处理步骤。

在整个模糊测试过程中通常仅用一个命令行选项（即所有输入都基于该选项执行），因为不同的选项代表了不同的代码覆盖，而一次全面的测试需要列举所有的选项，因此一个高效的方案便是若对于一个选项而言当前输入无效，则跳过剩余的所有选项。

#### 5.1.2 Deep Learning Systems

测试深度学习系统（DLS）的过程类似于测试命令行，通过生成输入（可以是训练数据、测试数据或不同目标上的深度学习模型）测试 DLSs 以获得更好的 fitness（可以是神经元覆盖率、损失函数、运算符级覆盖率），同时除了检测缺陷以外也会检查模型的健壮性。

#### 5.1.3 Operating System Kernels

OS kernel 包含了许多中断与内核线程，其执行状态无法确定，由此我们使用 hypervisor（如 QEMU）来运行内核，并通过 [Intel’s Processor Trace](https://zhangtong16.github.io/2019/06/05/Intel-Processor-Trace/) （PT）技术来获取代码覆盖；尽管这种方法能带反馈地测试不同种内核，但仍需要人工构造语法&语义正确的输入。

因为输入包括文件系统镜像或一系列系统调用，fuzzers 可以以更轻量级的方法进行测试：在系统调用的数据依赖项被分析/推断出来后生成一系列系统调用并在目标内核上运行，并监测代表潜在漏洞的 system panics。

另一种测试方法是通过模拟外设并生成相应输入来测试内核驱动。

#### 5.1.4 Cyber-Physical Systems

**信息物理系统**（Cyber-Physical Systems，CPS）包含两个紧密结合的主要成分，即**计算元素**（computational elements）与**物理过程**（physical processes）。

一个被广泛使用的计算元素是**可编程逻辑控制器**（programmable logic controller，PLC），其控制着物理过程的驱动器并从传感器中获取输入，因此在 fuzzing CPSs 时 fuzzer 可以替换掉 PLCs 并通过网络直接向驱动器发送大量的命令。

PLC 的二进制文件也是 CPSs 的一个可测试点，但其有着多种二进制格式以及复杂的与物理实体间的通信；基于对 PLC 二进制文件与开发平台的分析，自动化的 fuzz 可以在其运行在 PLC 设备上时进行。

#### 5.1.5 Internet of Things

IOT 的自动化 fuzzing 包括固件模拟与网络级测试：

- 模拟器可以在没有对应的硬件时模拟运行 IOT 固件，以灰盒模式测试目标程序。
- 网络级的 fuzzing 以黑盒模式进行测试，即通过网络向 IOT 设备发送信息，以响应作为执行结果，fitness 便是类型数量。

#### 5.1.6 Applications with Graphical User Interface

GUI 程序的执行比命令行慢得多，而执行速度是 fuzzing 的关键，因此对 GUI 程序的自动化测试通常将 GUI 替换为一种更快的方案并以命令行模式执行目标。

> 例如对 UI 操作建模后为安卓应用生成事件序列（event sequences）

此外，fuzzer 也可以使用 **hardness** 来准备执行上下文，以直接唤醒 GUIs 中的目标函数。

#### 5.1.7 Applications with Network

智能合约、协议实现、云服务、Android Native System Services、机器人装置等通过网络接收输入，由此可以在本地生成输入后由目标应用远程执行，自动测试的效率依赖于生成输入的质量与反映执行状态的 fitness。

### 5.2 Automatic Detection of Bugs

对于漏洞**检测器**（detector）而言漏洞的代码区域不可知，甚至不知道程序中是否存在漏洞，因此在自动 fuzzing 中记录潜在漏洞就变得十分重要，漏洞的**标志**（indicator）通常是程序执行时崩溃，也有一些基于漏洞**模式**（pattern）设计的专一而高效的指示器。

本节主要介绍成功由 fuzzing 发现的六种漏洞：内存损坏、并发漏洞、算法复杂性、spectre 型漏洞、测信道、整型漏洞。

#### 5.2.1 Memory-violaton Bugs

**内存破坏漏洞**（Memory-violaton Bugs）是最古老也最严重的安全漏洞，分为两类：

- **空间安全损坏**（spatial safety violation）：即非法内存访问。如 Fig.7a 便是一个越界（out-of-bound）内存访问。
- **时间安全损坏**（temporal safety violation）：即非法内存引用。如 Fig.7b 便是一个 use-after-free 漏洞。

<img src="/assets/Pasted%20image%2020250630231526.png" style="zoom: 80%;" />

尽管已经有一些针对内存破坏型漏洞的缓解措施（migration），但由于开销、兼容性、健壮性等原因，大部分缓解措施并未实际被使用。

论文给出了两个例子：

- [Dowser](https://www.usenix.org/conference/usenixsecurity13/technical-sessions/papers/haller) ：认为缓冲区溢出主要发生于循环中对数组的访问，通过排序循环中内存访问指令并给更高排序的输入高优先级后利用污点分析与混合执行求解选中输入的路径约束以检测 OOB 漏洞。
- [UAFL](https://dl.acm.org/doi/10.1145/3377811.3380386)：由于 UAF 漏洞通常是分配→释放→重用三步走，这一漏洞模式驱动 UAFL 生成能够逐渐覆盖一整列潜在 UAF 漏洞的输入，潜在 UAF 序列通过基于漏洞模式的静态类型分析完成。

#### 5.2.2 并发漏洞

**并发型漏洞**（Concurrency Bugs）在程序没有合适的同步机制或运行顺序时发生，通常可以分为：

- **死锁型漏洞**（deadlock bugs）：等待资源释放（如锁）
- **非死锁型漏洞**（non-deadlock bugs）
    - 原子性损坏型（atomicity-violation）：破坏了某一代码区域的**期望序列性**（desired serializability），如 Fig.8a 所示的`Thread 1`第三行释放了`p->info`，`Thread 2`的第二行将`p->info`置为 NULL，从而引发错误。
    - 顺序型（order）漏洞：以错误的顺序对内存区域进行访问，如 Fig.8b 中`Thread 2`在`mThd`被初始化前对`mState`赋值，这会造成未初始化变量引用漏洞。

<img src="/assets/Pasted%20image%2020250630231624.png" style="zoom: 80%;" />

发现死锁漏洞的一个方法是在**锁顺序图**（lock order graph）上检测代表死锁的**环**（cycles）。

论文举了这些例子：

- [MagicFuzzer](https://ieeexplore.ieee.org/document/6227156/)： 为了提高效率，其会移除不在任何环中的锁，并检查剩余的环
- [ATOMFUZZER](https://dl.acm.org/doi/abs/10.1145/1453101.1453121)：对于原子性破坏，其会观测原子块内的锁被两个线程重复请求与释放的漏洞模式
- [CalFuzzer](https://dl.acm.org/doi/10.1145/1321631.1321679)：过多的线程交错（interleaving）带来状态爆炸（state-explosion），其基于交错的等价性缓解状态爆炸。

#### 5.2.3 Algorithmic Complexity

**算法复杂性**（Algorithm Complexity，AC）漏洞是算法在最坏情况下会显著的降低性能，从而可能导致**拒绝服务**（Denial-of-Service，Dos）攻击，Fig.9 展示了一个有着不同算法复杂度的例子，在最坏情况下可以被攻击者用作 DoS 攻击：
<img src="/assets/Pasted%20image%2020250630231752.png" style="zoom: 80%;" />

论文举了以下例子：

- [SlowFuzz](https://dl.acm.org/doi/10.1145/3133956.3134073)：通过生成增加执行指令数量的输入来发现 AC 漏洞；
- [HotFuzz](https://www.ndss-symposium.org/ndss-paper/hotfuzz-discovering-algorithmic-denial-of-service-vulnerabilities-through-guided-micro-fuzzing/)：通过最大化单个方法的消耗来检测 Java 中的 AC 漏洞；
- [MemLock](https://ieeexplore.ieee.org/document/9284141/)：通过边覆盖率与内存消耗来检测 AC 漏洞；
- [Singularity](https://dl.acm.org/doi/10.1145/3236024.3236039)：基于**最坏表现输入** （ performance input，WPI）总是遵循某种特定模式来合成输入生成程序。

#### 5.2.4 Spectre-type Bugs

**幽灵型漏洞**（Spectre-type Bugs）是一种利用**错误分支预测**（mispredicted branch speculations）来控制内存访问的微架构攻击，例如在 Fig.10 中攻击者可以利用有效值来训练分支预测为真，随后给变量一个 OOB 的值，此时预测器便会错误预测分支行为，从而错误地执行了第3、4行代码，造成了越界读取。

<img src="/assets/Pasted%20image%2020250630231810.png" style="zoom: 80%;" />

论文给出的例子：[SpecFuzz](https://www.usenix.org/conference/usenixsecurity20/presentation/oleksenko)。

#### 5.2.5 侧信道

**侧信道漏洞**（side-channel）通过对系统的非功能性表现（例如执行时间）来泄露信息，例如通过分支执行时间判断执行的分支。

> a3 师傅给了一个例子：[prefetch side-channel attack: bypassing SMAP and KASLR](https://gruss.cc/files/prefetch.pdf)

**JIT-induced side channels**（~~不懂咋翻~~）是一种由即时优化（Just-In-Time Optimization）导致的特殊侧信道，类似于幽灵型漏洞，通过训练 JIT 编译器优化单一分支以使得两执行分支间执行时间差大到可以被观测到。

#### 5.2.6 Integer Bugs

**整型上溢/下溢**（Integer Overflow/Underflow）在算术表达式的值超过机器类型所决定的范围时发生，或是在整型间转换时发生（比如 int to uint）

> 论文举了这个例子：[SmartFuzz](https://dl.acm.org/doi/10.5555/1855768.1855773)

### 5.3 Improvement of Execution Speed

执行速度对模糊测试而言非常关键，更高的执行速度意味着在同一时间能跑更多测试用例，从而提高发现缺陷的机会。

论文给出了这个例子：[SmartFuzz](https://dl.acm.org/doi/10.5555/1855768.1855773)。

#### 5.3.1 二进制分析

**静态插桩**（static instrumentation）是主流的获取执行状态的方式，因为其为 fuzzing 提供了更高的执行速度。

- 对开源程序而言，一个被广泛使用的静态分析工具是 LLVM，其在编译期进行插桩；
- 对于闭源程序而言，fuzzer 被限制于二进制分析，但二进制插桩工具有着不菲的运行时开销。

论文给出了这些例子：

- [RetroWrite](https://ieeexplore.ieee.org/document/9152762) 使用基于**可重汇编的汇编**（reassembleable assembly）的静态二进制重写技术，其关注于使用 64 位的**地址无关代码**（position independent code，PIC）的重定位信息来插桩汇编程序；
- [FIBRE](https://www.usenix.org/conference/usenixsecurity21/presentation/nagy) 通过四个修改**中间表示的阶段**（IR-modifying phase）来流水线化插桩；
- [STOCHFUZZ](https://ieeexplore.ieee.org/document/9519407) 通过多次重写来解决此前重写的遗留问题。

#### 5.3.2 Execution Process

执行速度同样可以在模糊测试过程中提升，例如 [UnTracer](https://ieeexplore.ieee.org/document/8835316) 观测到大部分测试用例并不会带来新的覆盖率，由此其仅追踪会增加覆盖率的测试用例。

论文出来了这些例子：

- [CSI-Fuzz](https://ieeexplore.ieee.org/document/9139349/) 使用边覆盖率来改进 UnTracer，因为块覆盖率丧失了执行状态信息；
- [Zeror](https://ieeexplore.ieee.org/document/9286017) 通过在 UnTracer 插桩与 AFL 插桩间切换来改进 UnTracer。

对于混合模糊测试，混合执行被用以求解路径约束，但符号执行在表示路径约束上较慢，[QSYM](https://arttnba3.cn/2022/10/30/PAPER-0X00-FUZZING_A_SURVEY_FOR_ROADMAP/) 通过移除一些耗时的内容（IR 翻译、快照等）来缓解性能瓶颈。

论文给出了这些例子：

- [Intriguer](https://dl.acm.org/doi/10.1145/3319535.3354249) 观测到 QSYM 仍求解不必要约束导致的性能评价，因此其使用符号执行由动态污点分析确认的更相关指令。
- [Xu](https://dl.acm.org/doi/10.1145/3133956.3134046) 发现 AFL 在并行跑 120 核时显著变慢，故其设计了新的操作原语（operating primitives）来提升执行速度

#### 5.3.3 各种应用

模糊测试被用以检测多种目标中的缺陷，如 IOT、OS Kernel VMM 等，需要根据目标特性进行客制化（Customization 音译）。

论文给出了这些例子：

- FIRM-AFL 通过结合用户态模拟与全系统模拟来缓解传统 IOT 固件 fuzzing 中全系统模拟带来的虚拟地址与内存访问间翻译及模拟系统调用的开销
- Schumilo 设计了一种[客制化 OS](https://www.ndss-symposium.org/ndss-paper/hyper-cube-high-dimensional-hypervisor-fuzzing/) 与[快速快照存储机制](https://www.usenix.org/conference/usenixsecurity21/presentation/schumilo)。

>**Gap 3**：对应用的自动化执行基于对其的深入理解，在设计自动记录安全缺陷的 indicators 时需要首先研究这些缺陷的特性


## 0x06 Fuzzing 未来研究方向

Fuzzing 未来的研究方向与挑战：

- **更灵敏的适应度函数（More sensitive fitness）**：研究人员意识到仅依靠代码覆盖率在发现复杂漏洞时存在局限性，因此尝试引入基于漏洞分析获得的信息来改进适应度函数。未来工作可聚焦于基于漏洞特性进行漏洞检测，尤其是针对现有模糊测试未能发现的漏洞进行深入分析。
  
- **更完善的模糊测试理论（More sophisticated fuzzing theory）**：目前大部分研究集中于种子调度，少数关注模糊测试的其他环节。构建完整的数学模型虽不易，但多种模糊测试过程可共存，例如利用博弈论同时考虑种子调度与字节调度。未来更宏大的研究方向包括探索模糊测试理论的局限性（如灰盒模糊测试的限制），以及结合多种适应度函数，设计既考虑漏洞出现又兼顾状态转移的先进模糊测试过程。
  
- **科学的评估方法（Sound evaluation）**：部分工作关注评估的可靠性，但尚无定论（见 §3.6）。亟待解决的问题包括：评估语料库应使用真实漏洞还是合成漏洞？静态测试能否作为区分模糊测试技术的最终标准？合理的时间预算如何确定？如何在缺乏可比较模糊测试器的情况下评估特殊目标（如硬件）？
  
- **可扩展的输入格式推断（Scalable input inference）**：利用格式或数据依赖项可显著提升模糊测试效率（见 §4.6 和 §4.7）。静态分析广泛用于格式和数据依赖推断，但其结果高度依赖特定程序且实现复杂。动态分析主要关注格式推断，较少涉及数据依赖推断，但具有更好的可扩展性。
  
- **高效的变异操作符（Efficient mutation operators）** 几乎所有模糊测试工具都采用混合变异器，但通常不动态调整变异器。部分研究优化了变异器调度，但鲜有关注可变变异器设计。鉴于变异器调度与字节调度密切相关，未来可探索基于字节调度的变异器设计，特别是在处理高度结构化输入时的调度策略。
  
- **更多类型的应用场景（More types of applications）**：由于应用复杂性，模糊测试在检测某些应用（如 CPSs）时能力有限。鉴于执行速度对模糊测试的重要性，提升难以被模糊测试的程序的执行效率是潜在的发展方向。
  
- **更多类型的漏洞（More types of bugs）**：模糊测试在检测内存破坏、并发漏洞、算法复杂性漏洞等方面表现良好（见 §5.2），但在权限提升、逻辑漏洞等方面仍有挑战。设计合适的漏洞检测指标（indicator）尤为困难，需要研究人员同时具备对模糊测试技术和目标漏洞的深刻理解。
