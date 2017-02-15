---
layout: post
title:  "MIT6.828 Lab1 Booting a PC Part 3"
tags: [MIT6.828, Operating System, 操作系统, 学习历程 , kernel]
comments: true
description: "深入JOS的内核细节"
keywords: "OS,kernel"
date:   2017-02-15 14:40:22 +0800
---


## 实验一大纲
- 环境配置
- 第一部分：PC Bootstrap
    - x86汇编
    - 模拟x86
    -  PC的物理地址空间
    -  ROM BIOS
- 第二部分：The Boot Loader
    - 加载内核
- 第三部分：The kernel


## 第三部分 The kernel

这一部分，我们会更加深入JOS的内核细节，内核代码先有一些汇编语言和一些设置以便C代码能正确地执行。

**Using virtual memory to work around position dependence**

当观察上面boot loader的链接地址和加载地址时发现它们都完美地匹配，但是在内核的链接地址和加载地址上就会有很悬殊的差异。回顾它们确保都明白我们说的。（链接内存要比boot loader要复杂得多）。

操作系统的内核经常有可能被连接和运行到非常高的虚拟地址上，比如0xf0100000，为了能够给用户级程序让出处理器虚拟地址空间。在下次实验中这个过程会很清晰。

然而大多数机器在0xf0100000上不会有物理内存，因此我们不能将内核储存在这儿。那么怎样储存呢？
我们会用处理器的内存管理硬件去map虚拟地址0xf0100000（链接地址，内核代码预计去运行的）到物理地址0x00100000（这儿，bootloader将内核加载到物理内存）。这样的话，即使内核的虚拟地址足够高得来给用户进程腾出大量的地址空间，它还是会加载在PC RAM的1MB处的物理内存。该方法需要PC有至少几M的物理内存（以便物理地址0x00100000有效），在1990后的PC都是可以实现的。

在下一个实验中，我们会将整个地下的256MB的PC物理地址空间（从0x00000000到0x0fffffff）分别映射到虚拟地址0xf0000000到0xffffffff。这样一看就知道为什么JOS只能用最开始的256MB的物理内存了。


这里我们仅映射4MB的物理内存，这也足够我们运行了。我们利用在kern/entrypgdir.c的手写的静态初始的页目录和页表。直到kern/entry.S设置了CR0_PG 标志，内存引用就视为物理地址（严格说来，它们是线性地址，但是boot／boot.S设置了一个identity来映射线性地址到物理地址并且我们也不会去改变）。一旦CR0_PG被设置好，内存引用虚拟地址，这虚拟地址是由虚拟地址硬件到物理地址的。entry_pgdir将虚拟地址0xf0000000到0xf0400000 转到物理地址0x00000000到0x00400000, 同时虚拟地址0x00000000到0x00400000到物理地址0x00000000到0x00400000。任何不在这两个范围的虚拟地址将会造成硬件异常，将会造成QEMU出错和退出。


>练习7:使用Qemu和GDB去追踪JOS内核文件，并且停止在movl %eax, %cr0指令前。此时看一下内存地址0x00100000以及0xf0100000处分别存放着什么。然后使用stepi命令执行完这条命令，再次检查这两个地址处的内容。确保你真的理解了发生了什么。如果这条指令movl %eax, %cr0并没有执行，而是被跳过，那么第一个会出现问题的指令是什么？我们可以通过把entry.S的这条语句加上注释来验证一下。


那么设置断点0x100000c，然后```si```到出现```movl %eax, %cr0```,查看内存得到发现：在这条指令发生之前，两地址值不一样，再次```si```两处到值是一样的。
这说明存放在0xf0100000处的内容，已经被映射到0x00100000处了。

![friends](https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/lab1_exe7.PNG?raw=true)



**未完**







