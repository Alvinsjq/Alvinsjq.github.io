---
layout: post
title:  "如何看懂Polyspace的分析结果并进行审查"
tags: [Polyspace,Operating System Design,Static Analysis]
comments: true
description: "最近实验室老师让熟悉一下一个静态分析代码工具Polyspace，Polyspace用于静态测试运行时错误，主要进行非功能性测试，这篇文章主要是根据官方的文档给出如何看懂Polyspace的分析结果，从而为下一步纠正代码缺陷提供依据和思路。"
keywords: ""
date:   2017-04-04 14:40:22 +0800
---
首先是需要对分析结果所显示的窗口有一定的了解，能够得到这些图标中的信息。

## Dashboard
在运行完分析之后，会出现一些可视化的图表，用图形的形式为分析结果提供了一些数据。那么首先解读一下这些图标数据。

### Code covered by analysis

![Code covered by analysis](https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/Polyspace/results_statistics_code_covered_by_analysis.png?raw=true)

很容易看出，这张图主要是讲了分析所覆盖的代码比例，从中可以看出：

- Files：分析过的文件占总文件的一个比例。如果一个文件包含了编译错误，那么Polyspace Bug Finder不会分析这个文件。
- Functions：在已分析的文件中，分析过的函数占总函数数量的比例。如果一个函数所用的分析时间大于一定的界值，那么Polyspace Bug Finder不会分析该函数。

### Defect distribution by category or file

![Defect distribution by category or file](https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/Polyspace/results_statistics_defect_distribution.png?raw=true)

这是对检测出的代码缺陷的一个分类图，从这张图表可以得到最多（少）的10种错误，它会有两种呈现方式，一种是按缺陷类别进行排序，另一种是按缺陷文件进行排序。在第一种中将光标移至红条处可以看到是哪个文件的，在第二种中将光标移到红条处可以知道是哪些类型的缺陷。

这样子，就可以根据图表对缺陷最多（少）的类型（或文件）优先排查和修复。

这里是一个对有影响的缺陷的一个[级别分类](https://cn.mathworks.com/help/bugfinder/ug/result-grouping-by-impact.html)。

###  Coding rule violations by rule or file 

![Coding rule violations by rule or file ](https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/Polyspace/results_statistics_coding_rule_distribution.png?raw=true)

针对所写的代码是否违背所检查的代码规则，例如MISRA®, JSF® 或自定义的，那么这张图就包含了这些规则违背。

同样可以按照违反类型以及文件来查看最多（少）的违反的规则，对于具体的那些规则这里有：

- [Supported MISRA C:2004 and MISRA AC AGC Rules](https://cn.mathworks.com/help/bugfinder/ug/misra-c-coding-rules.html#brjxmed)
- [Supported MISRA C++ Coding Rules](https://cn.mathworks.com/help/bugfinder/ug/misra-c-coding-rules-1.html#bse_zo6-1)
- [Supported JSF C++ Coding Rules](https://cn.mathworks.com/help/bugfinder/ug/supported-.html#bru7_x0-3)


## Result List
在result list界面展现了一歇结果的参数，可以有不同的排列方式：

- None：对排列代码缺陷和代码规则违反不做要求。根据严重性默认排列。
- Family：根据不同的缺陷类别进行排列。详细的缺陷类别可见[Bug Finder Defect Groups](https://cn.mathworks.com/help/bugfinder/ug/description-of-check-categories.html)。
- Class（only C++）：根据类排列，在每个类里再根据方法排列。
- File：根据文件排列，在每个文件中再按照函数排列。

那么它们的参数是哪些呢？又下表可知：

|参数|描述|
|---|---|
|Family|结果属于哪一个群类|
|ID|结果唯一的标志符|
|Type|代码缺陷或者是规则违反|
|Group|结果的类别，例如对于代码缺陷可以是静态内存、数值的、控制流、并发等等；对于代码规则违反可以是 MISRA C®中的某一条规则|
|Check|结果名，例如对缺陷而言就是缺陷名；对代码规则违法而言就是代码规则编号|
|File|出自哪个文件|
|Class|出自哪个类|
|Function|出自哪个函数|
|Folder|源文件所属的文件夹|
|Serverity|结果的严重级别：High、Medium、Low、Not a defect|
|Status|审查结果的状态，可能的状态：Fix、Improve、Investigate、Justified、No action planned、other|
|Comment|注释|


这样就可以通过这些结果来[审查和修复结果](https://cn.mathworks.com/help/bugfinder/ug/review-and-comment-results_bty8g0k-1_1.html)。


## Source
![source code](https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/Polyspace/source_pane.png?raw=true)

这个窗口可以看到标有颜色的代码缺陷的源代码，右击它们会出现一些菜单选项。

## Result Details

![Result Details](https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/Polyspace/check_details.png?raw=true)

这和界面包含了对一个缺陷的详细的信息。得到它只需要在Result List中选择特定的缺陷。在2017a中的PolySpace版本中，还可以对每一个check分配严重等级和状态以避免被重复check。

- 右上角展现了包含这个defect的文件及其函数；
- 黄色框中展现了这个defect的名字以及解释了为什么这个defect会发生；
- Event列出了导致这个defect的代码指令，Scope列出了包含这些指令的函数，如果这个指令不在一个函数中，那么就显示它所在的文件；
-  Variable trace可选项允许你看见与该defect相关的其他的指令。


那么如何审查和修补结果呢？那么接下来的例子就是说明怎样去审查和注释你的Bug Finder结果。当你在审查结果是，你可以对这些defect安排一个状态，以及输入一些注释来描述你审查的结果。

- 首先打开Result List选择一个defect，然后关注Result Detail窗口中的信息；
- 进一步审查这个结果，确定是否要修改你的代码，或在之后再审查，或者保留这个代码但是提供一些说明；
- 在Result List或Result Detail上提供以下的审查信息：
    + 严重性 
    + 状态
    + 注释
- 如果要对多个结果一起审查并提供信息，那么就需要同时选择这些结果；
- 最终记得保存。

当然还可以从之前的分析中导入审查注释。

对一些无法修复的代码，可以添加一些注释，注释最好以这种格式出现，详情可见[Annotate Code for Known or Acceptable Results](https://cn.mathworks.com/help/bugfinder/ug/annotate-code-for-known-results.html)


## 参考资料

- https://cn.mathworks.com/help/bugfinder/ug/overview-of-results-manager.html




