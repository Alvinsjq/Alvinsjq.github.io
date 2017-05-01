---
layout: post
title:  "Simulink／Stateflow上的一个整合的测试用例生成工具 - 论文笔记"
tags: [Simulink,Test Generation]
comments: true
description: "最近实验室又有新的研究任务，这次与Simulink/ Stateflow 模型的测试用例自动生成相关，与传统的软件测试中的测试用例相同，这里也有覆盖率这一标准，那么如何提高模型的测试用例的覆盖率呢，这篇博文作为相关论文的阅读笔记，了解了一个生成工具...."
keywords: ""
date:   2017-05-01 13:40:22 +0800
---

最近实验室又有新的研究任务，这次与Simulink/ Stateflow 模型的测试用例自动生成相关，与传统的软件测试中的测试用例相同，这里也有覆盖率这一标准，那么如何提高模型的测试用例的覆盖率呢，这篇博文作为相关论文的阅读笔记，了解了一个生成工具....

首先了解一下几种该领域的测试用例生成技术以及一些有名的工具。
## 测试用例生成技术
对于Simulink/ Stateflow diagram的测试生成和验证有许多不同的方法。下面对这些方法进行一些梳理。

### Regression testing or randomised test generation （回归测试或随机化测试生成）

这类方法是比较流行的一类方法，例如Reactis工具就是一个产生随机的Simulink/Stateflow模型的测试的工具。atpathy et al. 则是从Simulink/Stateflow中基于离散随机测试并且利用一系列模式引导的启发式非线性块来产生测试用例。Oh et al利用遗传学算法为Simulink/Stateflow提出了一种转移覆盖率测试。虽然在随机测试上的技术有一定的发展，但是还是很难得到一个接近全覆盖的测试，并且大量的启发算法也许还会阻碍大量Simulink块的生成。

### Model-checking based automated test generation approach （基于自动模型检查的测生成技术）

该技术将测试对象的否定的反例作为测试用例。例如Mathworks的Simulink Design Verifier工具就提供了基于自动模型检测测试生成技术为Simulink/Stateflow模型生成测试用例，并限制一些块不能被分析，以及只能够实现块层次的覆盖。完成转换Simulink/Stateflow到形式语言的实验室叫做符号分析实验室（Symbolic Analysis Laboratory - SAL），他们是基于模型检测实现的。除此之外还有很多团队，例如Marre and Arnould、Sofronis and co-workers、Miller et al等，都用到了Lustre。Simulink Design Verifier用一个叫做Prover，但它对于生成非线性方程的测试用例有一定的缺陷。微软的Z3是一个能解决非线性方程的成熟的证明器。

### Constraint-solving based automated test generation approach（基于解决限制的自动测试生成技术）

该技术应用在 Gotlieb et al.上。而对于由NASA开发的Symbolic PathFinder，结合了符号执行和解决限制来进行测试的自动生成。利用该技术的流行的工具还有CPLEX , MATLAB , Maple 等。

### Mutation-based test generation approach（基于突变的测试生成方法）
该技术引入错误并在产生的测试用例中展示错误。


当然还有一些混合的方法技术，那么在这一领域中比较重要的商业工具有：

- 来自Mathworks的Simulink Design Verifier (SDV)
- 来自Reactive Systems Inc的Reactis
- 来自T-VEC Technologies的T-VEC 
- 来自Honeywell International的HiLiTE


## 介绍

在汽车和航天航空产业中，开发控制软件的主要的建模方法就是Simulink/Stateflow (SL/SF)。在基于模型的测试中，由一个设计模型而得到的测试用例通常可以用来体现模型代码的一致性。像在一些安全标准中就会要求这样的基于模型的测试，例如ISO 26262，这样一来可以体现出软件的符合性。而这篇论文用的测试工具能够帮助得到更好的覆盖能力的测试用例。这个工具叫做SmartTestGen，它整合了不同的测试生成技术。因此这篇论文就是讨论这个工具以及那些不同的测试技术——随机测试、限制解决、模型检查和启发式方法。这些在上面一节中也提到了一些。然后用了20个产品模型作为例子，对比了其他两个商用的测试生成工具。

那么这个整合了不同生成测试引擎的测试工具，自然是每个引擎利用了一种测试技术。这个工具的使用了一个特别的序列，以便成本低的目标可以早地被一个成本较低的引擎覆盖。

这个工具的蛀主要创新点是：

- 设计和实现了这样的一种整合型的测试生成工具，这些不同的生成测试引擎都是由它的团队开发的。
- 评估了该工具的原型，跑了20个工程模型，得到的结论就是性能要好于其它两个商业的测试生成工具。

## 不同的生成测试引擎

来自Reactive Systems Inc的Reactis、来自BTC的Embedded Tester以及Mathworks的Simulink Design Verifier (SDV)是一些商用的自动生成测试用例的工具。Reactis tester结合了随机测试与引导仿真技术。Embedded Tester中，首先会将模型导入TargetLink代码生成工具，然后分析生成的C代码产生测试用例。而SDV的主要功能就是证明SL／SF模型的特性，该工具还能够展现模型元素的不可达性。

AutoMOTGen是一个非商业的基于模型检测技术的测试生成工具，REDIRECT是一个集随机测试、定向自动随机测试DART、混成具体与符号测试于一体的测试生成工具。（note：这里翻译的词不一定很准确）

接下来主要讨论SMARTTESTGEN这个工具中用到的生成技术


### A 基于模型检测的测试生成技术

将SL／SF的子系统模型转换为SAL[1]模型，记住在SL／SF模型上的覆盖标准，SAL模型带有一些诱导变量，一个诱导变量的可达性体现了一个模型元素的可达性；可达的轨迹就变成了这些测试用例[2]。这个技术同样可以用来证明这些模型元素的不可达性。



### B 随机测试技术

给定输入类型和它们的范围，模型产生随机测试序列并进行仿真检查它们是否覆盖模型元素。每一个序列的大小和数量会很明显地影响着模型的覆盖率。


### C 局部限制解决

为了在模型中能够覆盖例如一个决策分支点处的目标，我们可以对在该目标上的变量做一小部分向后的切片来确定相关的区间、外面的输入变量以及获得能够被解决去寻找一个测试序列的一个限制。限制解决也能够于随机测试混合。

下图a中展现了带有分离的integrator的Simulink模型。xi代表的的是变量x在时间点i时的值（i>=1），假设初始化integrator的值，例如令x0为0，并且输入范围为0..2，那么我们就有以下等式：对于i>=1,xi = xi-1 + ai;当 xi>=100,yi = true 否则就是false。假设我们以一个随机的输入序列[a1 , . . . , a100 ]来仿真实验，结果我们有 x100 = 90 并且 y100 = 
false。在之前的随机序列之后再立即使用符号序列。[a1‘，..，a10’]，我们可以得到一下限制条件：

```
x′1 = x100 +a′1 ∧ x′2 = x′1 +a′2 ∧ ... ∧ x′10 =x′9 +a′10∧ (x′1 ≥100 ∨ .. ∨ x′10 ≥100)
```

要使以上的限制条件满足，那么我们就得到[a1‘，..，a10’]的值。假定我们将这些符号值替换成具体的值，那么新的测试序列[a1, . . . , a100]#[a′1, . . . a′10]  得使得输出y为true，这里的#是指串连符号。那么这里就结合了随机与限制的解决方式来覆盖目标。由于我们并没有在初始值时就使用限制解决，而是在测试串上的某个位置之后开始使用，我们就称之为局部限制解决。

![Example models to illustrate local constraint solving and heuristics](https://github.com/Alvinsjq/6.828_tasks/blob/master/paper_data/1_simulink.png?raw=true)


### D 基于启发式引导覆盖

上图b中，表示的是一个Simulink的系统，S1、S2是其子系统。假设一个随机的输入序列 [(a1, b1), . . . (a10, b10)] 使得 x = true，或者也可以是一个通过限制的一个序列。类似的，假设存在一个测试序列 [(a1 ‘, b1 ’), . . . (a10‘ , b10’ )]使得 y = true。由图可以观察到计算x和y的输入集是不相交的。因此我们应用启发式得到输入序列[(a1, b1‘), . . . (a10, b10’)] 而该序列使得AND块的输出为true。
这里的启发式方法是一系列分析目标的模式并且唤起适当的启发式方法来找到测试序列分别覆盖不同的目标值，更多关于启发式的细节可以参考[3]。


## SMARTTESTGEN 的实现

![SmartTestGen Tool Architecture](https://github.com/Alvinsjq/6.828_tasks/blob/master/paper_data/2_simulink.png?raw=true)

上图展示的是该工具的一个整体框架，将一个SL／SF模型以及一个测试详细说明作为输入，然后输出一套测试用例。它使用了我们上面介绍的4个测试生成引擎。STGen有三个主要组件：

- 集中控制信息表
- 测试生成引擎集合
- 管理监督模块

管理监督模块唤起适当的测试用例生成引擎，接受由不同引擎生成的测试用例，并且更新集中控制信息表。下面介绍一下这几个组件。

### 集中控制信息表 （Centralized Information Table）

<img src="https://github.com/Alvinsjq/6.828_tasks/blob/master/paper_data/3_simulink.png?raw=true" width="500" >

这个控制信息表包含了能够唤起有效测试生成的测试生成引擎的所有数据。这张表由管理监督模块维护，下表就是它的基本结构。

 第一列是块处理，在模型中每个块有独一无二的标号；这些块中包含覆盖的候选数据。接下里的三列分别是决策(D),条件（C）以及 MC/DC 覆盖。表中的每一个块都包含许多与目标相关的覆盖点。例如，逻辑与关系块只由决策覆盖测量，并且每个只有两个目标。第5列包含了特殊块的路径；一条路径是一个字符串等同于特殊块的位置。一条路径表明了块的嵌套级别，同时也辨别出用什么技术来覆盖这个块。第7列列出了覆盖相关块的一系列测试用例。

给定一系列在模型中的SL/SF块，和它们的内容，一个分类算法可以根据数据估计哪个测试生成引擎可以覆盖哪个目标；这种规则是从经验中得知的，用户也可以选择无视这样的规则，第6列存储了这些信息。

### 测试生成引擎集合 （Test generation engines）

在STGen中，我们使用了三个测试生成引擎，它们实现了之前所提到的四个测试生成技术。

- 随机测试生成引擎 （random test generation engine）
    基于输入的类型和范围生成随机的输入序列
- 模型检测引擎 （model checking engine）
    基于SAL的模型生成测试引擎，利用模型检测技术覆盖给定的目标
- 限制解决与启发式引擎 （onstraint solving and heuristics engine） 
    利用限制解决与启发式方法拓展给定的测试用例来覆盖给定的目标

限制解决与启发式方法引擎整合在了一起，那是因为限制解决需要在启发式中被唤起。
默认的唤起的顺序是 —— 随机、启发、模型检测

### 管理监督模块 （Supervisor Module）

<img src="https://github.com/Alvinsjq/6.828_tasks/blob/master/paper_data/4_simulink.png?raw=true"  width="450">


上面也刚提到，这个模块通过选择不同的测试生成引擎来控制测试生成进程。上图是管理监督模块的主要活动。

- initializeCovTable()
该活动得到SL/SF模型和用户给定的测试详细说明，找到模型中的目标，并初始化信息表CI_Tab中的记录。

- instrumentModel()
接着该活动instruments模型，instrumentation被用来捕获与仿真运行相关的所有信息。根据外部输入和它们范围，接下来就能获得随机输入序列（测试套装TS1）。然后instrumentation模型就仿真这些覆盖目标的测试用例。

- updateCovTable()
该活动相应地更新CI_Tab。

- classifyBlocks()
填满CI_Tab的第6列。

- selctEngine()
自然地在表中还会有未覆盖的目标，那么该活动就会从未使用的引擎中选择合适的测试生成引擎。表中的第6列提供了这一选择的信息，最大可能覆盖做多目标的引擎就会被选择。然后这个被选中的引擎就会来覆盖这些目标。此时，当前的测试生成引擎就会产生一些新的测试用例(TS2)；然后已覆盖记录表与当前的测试套装就会相应地更新。

- reClassifyBlocks()
那么现在就已经有些目标已被覆盖了，那么表中第6列就要更新数据。然后进行下一次选择知道所有的测试生成引擎都被选择了。
那么到了最后模型检测引擎再一次被调用，弄清楚是否能证明那些未被覆盖到目标是不可达的。

最后我们就能够得到对于所有覆盖目标的测试用例套装，以及那些不可达的目标的集合。


## 结论

STGen是利用Matlab脚本语言 - m-script实现的。用了20个SL/SF设计的模型来在STGen上进行测试用例生成。对比了其它两个工具（Reactis 和 Embedded Tester），对比数据在原文中有清晰的图。那么最终的结果就是在覆盖上，相对来说有了更好的数据。不可达报告也提高了测试准确性。

## 参考资料

[1]SRI International.[SAL home page](http://sal.csl.sri.com)

[2]Gadkari A, Yeolekar A, Suresh J, Ramesh S, Mohalik S, Shashidhar KC.
AutoMOTGen: Automatic Model Oriented Test Generator for Embedded
Control Systems. Proceedings of the CAV08, 2008; 204-208.

[3]Satpathy M, Yeolekar A, Ramesh S. 2008. Randomized Directed Testing (REDIRECT) for Simulink/Stateflow Models, ACM/IEEE International Conference on Embedded Software (EMSOFT’08), Atlanta.

[4]原文 [“An Integrated Test Generation Tool for Enhanced Coverage of Simulink/Stateflow Models”](https://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=3&cad=rja&uact=8&ved=0ahUKEwiq5eXZrs7TAhVE_mMKHaYPBYoQFggzMAI&url=https%3A%2F%2Fpdfs.semanticscholar.org%2F769f%2Fa6ee770bc52fbdfe9c0cf785e91889223459.pdf&usg=AFQjCNGfFGigT06Bx9gI-tNR9MbZ5O2J-w&sig2=ggk-MhFxMBU0wZSlHK6NqA)





