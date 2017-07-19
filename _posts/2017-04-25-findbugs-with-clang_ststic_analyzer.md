---
layout: post
title:  "用Clang静态分析工具寻找bug"
tags: [clang,static analyzer,LLVM,编译原理]
comments: true
description: "这是Apple的一个presentation，题为Finding software bugs with the Clang Static Analyzer,演讲者叫Ted Kremenek."
keywords: ""
date:   2017-04-25 13:40:22 +0800
---

Clang 静态分析工具能够通过分析源代码自动寻找出其中的bug。

## 利用编译技术寻找Bug

### 编译时警告

通过clang一个代码文件可以得到一个编译警告，可以看到clang终端编译并详细地说明了代码在哪儿出现了问题。

### 静态分析

静态分析技术并不是软件测试的替代品，在这次的演讲中将提供对这个工具的一个上层的概述。
由于由编译器警告所展现出来的代码检查是有内在缺陷的，也就是说编译结果中的信息是有限的，甚至有的时候仅仅提供一些没有意义的信息。但是往往我们恰恰需要有关程序的更加具体而详细的错误信息，也就是说程序员们不单单对错误信息出现在哪里感兴趣，甚至是想知道什么类型的错误，例如，内存泄漏、缓冲过载、逻辑错误等等。

那么之前也许仅是对程序进行测试从而找到程序路径上可能犯下的错误，但这需要十分具体的测试用例以及测试相关人员丰富的测试经验。那么提到静态测试技术它有三个好处：

- 可以提前发现这些错误，因为如果错误遗留到之后的话，那时修改可能会对工程造成很大的影响，所以越是早发现这些错误，那么修改它们的代价也就越小；
- 与此同时静态分析能够系统地检查代码的每一个角落；
- 不需要测试就可以寻找到错误；静态分析特别擅长那些可能在实际代码中不怎么运行到的分支的错误，而往往这些是测试得相对不怎么完全的地方。因此不需要测试是非常强大和有用的。并且有些情况下的代码错误是很难通过测试发现的。那么静态分析就对那些很难用测试手段发现的错误有很好的效果。当然，静态分析绝不是代替测试，因为肯定会有许多静态分析无法检测出来的错误。


### This Talk: Clang “Static Analyzer”

#### Demo 
可以看另一篇[博文](https://alvinsjq.github.io/2017/install_clang_ststic_analyzer/)。

#### How does static analysis work?

例如这样的一个简单的函数：

```c
int f(int y){
    int x;

    if(y) 
        x=1;

    printf("%d\n",y);

    return x;
}
```


在函数f中x最先声明，那么如果y为0的话，我们实际上不能为x进行赋值，那么在编译器之后就会警告我们x可能没有初始化。那么具体里面发生了什么呢？
<!--more-->
[17/70]可以看下图，是这段代码的一个控制流图，意思就是说，在clang本质中，c代码可能看起来就像是这样子的。

[18/70]这里具体是怎么一回事呢？其实就是编译器看到了从顶部到底部过程中的这条边，然后在上面定义的x并没有被赋值，但在最后却要将x的值返回。

[19/70]因此这条路径呢就会发生我们所说的一个内存泄漏。

[21/70]那么如果现在要去修改这个错误的话，我们就得加上一个保护条件，

```c
int f(int y){
    int x;

    if(y) 
        x=1;

    printf("%d\n",y);

    if(y)             //add a guard
        return x;

    return y;
}
```


[23/70]现在的程序控制流图也稍微改变了一点，但是有趣的是，即使是这样，用编译器编译之后还是会出现警告：x可能没有初始化。

[24/70] 人们往往会忽视掉这样的一个警告，那么可能的原因就是在这样的情况下，其实这个信息并没有什么意义。（note：可以用分析器测试一下有没有warning！）那么具体是怎么回事呢？

[25/70] 就这里的控制流图而言，一种情况可能就是每一个if的分支都直接执行后面的语句(taken)，也就是执行右边的分支，那么肯定需要这里的y为0。

[26/70] 另一种情况就是当y不为0的时候（随便一个不为0的y都可以），就会走左边的分支，这样也是一条可行的路径。

[28/70] 那么问题就在于不管是gcc还是clang，提示warning的时候都是指的是这样的一条路径，显然这条路即要要求y为0，又要要求y不为0，显然是一条不可行的路径。而编译器提示的警告出现在了一条不可能的路径上。

### False Positives (Bogus Errors)
[32/70] 因此我们对这样的情况取了个名字叫做False Positives（Bogus Errors），可以翻译为假错误。它们可能会在各式各样的情况下发生，比如刚才展示的那条不可行的路径。而另一方面，不管你是调用的哪一种编译器，编译器分析程序总不是十分充分的，例如它是怎样从外面调用进来的等等，那么也许可以通过我们告诉编译器更多的信息来解决这个问题，但是最后我们也许只能做一个错误的假设；那么减少这样的一种false Positive的方法，其中一个就是要有更多的精确的分析，但其实还是很难完全消除这样的一种False Positives。

### Flow-Sensitive Analyses
[34/70]正常的编译器是做什么的呢？其实这里是有一个名字的，也就是叫做数据流-敏感分析，也就是说，编译器能够进行值的一些推理;

```c
     /*Flow-sensitive analyses reason about flow of values*/
     y = 1;
     x = y + 2;  // x == 3

```

[35/70]但是不能够从路径中推断出准确的值，就例如这样的一种情况，其实编译器并不能推理出++x时，x的准确值是多少。

```c 
    /*No path-specific information*/
    if (x == 0)
        ++x;     // x == ?
    else
        x = 2;   // x == 2
    y = x;       // x == ?, y == ?

```

[37/70]LLVM的静态单赋值(SSA)形式就是专为数据流敏感算法而设计的，而这样的好处是它们可以被用在优化和编译器警告上, 并且这是十分重要的，因为我们肯定想要我们程序能够在一定的时间内编译好。但是呢，在寻找bug上并不总是最好的。

### Path-Sensitive Analyses
[39/70]因此呢，我们这里所实现的叫做路径敏感分析（Path-Sensitive Analyses
），那么从条件分支上追踪信息，然后就能得到相对来说比较复杂的式子，例如说，下面这种情况，如果你从这样的分支中获得信息，那么最终需要判断x和y的值是不是都是1或者都是2。因此这里就取每一种可能的路径的析取。

```c 
    if (x == 0)
        ++x;        // x == 1
    else
        x = 2;      // x == 2

    y = x;          // (x == 1, y == 1) or (x == 2, y == 2)
```

[40/70] 那么如果用这样的方式再去跟踪之前的false positive的没有初始化的例子，路径敏感分析就只会选择2条路径，并且不会出现false positive的错误警告。


[41/70]而真正的问题就在于，一旦开始追踪代码中所有的独立的路径，你会受到很大的阻碍，并且当你开始执行循环，那么之后的问题可能变的更加复杂，但是我们还是有方法可以去解决的，例如路径合并等使得这个问题变的容易处理。

但是，在寻找错误的空间中，我们想要放弃两种东西，一个就是完整性，我们想要将其限制在工作之外，也就是我们不可能寻找出你的程序中的所有的bug，或者说可能不会走遍你代码中的所有的路径；而另一个就是需要回到刚才讲到的false positives的合理性，我们没有必要证明说你的程序是不存在bug的，因为这功能其实并不是一定要在分析工具中出现的，不过只是可以灵活地去选择它，因为这在编译中其实是没有必要做的，但是这儿的目标就是能够寻找到bug然后修正它们，而不是生成使得可执行机器语言。

[42/70]而在clang静态分析器中，这两种分析方法都使用到了。

### Checker Results
[60/70]那么这样的一个工具一开始是在Apple内部使用，在2008年的WWDC上发布。那么接着讲一些实现的细节。

### Why Analyze Source Code?
[61/70]那么究竟为什么分析源代码？如果分析LLVM IR? 其实虽然LLVM IR有很多信息，当然这些都仅仅是底层的信息，但同时也抹去了大量在高级语言中表达出的信息，另一方面我们是在找错误，实际上你们也只对高层次的信息感兴趣。总的说就是，我们关注的并不是生成机器语言，而是产生一些有意义的信息好能够让用户可以真的去使用来修理他们的问题。那如果用户不知道怎样修理，那就完全没有意义了。因此我们实际上需要大量源代码层次上的信息，例如说宏指令等，那么所有llvm层的信息都不必提供。


### Clang Libraries
[64/70]接着看一下Clang的软件库，首先是Analysis是为做一些数据流分析提供的一歇基本的解决方法，同时也是使用Rewriter来做一些HTML的诊断，这里会生成带有源代码的htmls，也就是插入一些html标签。



### 最后贴上一个总结：
**Source-level Control-Flow Graphs (CFGs) **

**Flow-sensitive dataflow solver**

- Live Variables
- Uninitialized Values

**Path-sensitive dataflow engine **

- Retain/Release checker
- Logic bugs (e.g., null dereferences) 

**Various checks and analyses**

- Dead stores 
- API checks


以上就是Apple早期对clang static analyzer的一个介绍的总结，由于自己也是在嘘唏，所以整理的不对的地方欢迎指正。


### 参考资料

- [2008 LLVM Developers Meeting - Finding Bugs with the Clang Static Analyzer 1 of 5](https://www.youtube.com/watch?v=4lUJTY373og&t=102s)
- [2008 LLVM Developers Meeting - Finding Bugs with the Clang Static Analyzer 2 of 5](https://www.youtube.com/watch?v=4lUJTY373og&t=102s)
- [2008 LLVM Developers Meeting - Finding Bugs with the Clang Static Analyzer 3 of 5](https://www.youtube.com/watch?v=4lUJTY373og&t=102s)
- [2008 LLVM Developers Meeting - Finding Bugs with the Clang Static Analyzer 4 of 5](https://www.youtube.com/watch?v=4lUJTY373og&t=102s)
- [2008 LLVM Developers Meeting - Finding Bugs with the Clang Static Analyzer 5 of 5](https://www.youtube.com/watch?v=4lUJTY373og&t=102s)
- [PPT的链接](https://www.google.com.hk/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwjyz52s48TTAhVPzGMKHRSFCo8QFggiMAA&url=http%3A%2F%2Fllvm.org%2Fdevmtg%2F2008-08%2FKremenek_StaticAnalyzer.pdf&usg=AFQjCNEGUpaoyYb3MFI7ZlOlDoPFSVhiEg&sig2=un7J97TXKHCfi_IhXddU3Q)





