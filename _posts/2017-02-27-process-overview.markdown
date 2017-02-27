---
layout: post
title:  "关于进程、页、地址空间"
tags: [MIT6.828,OS,process]
comments: true
description: "MIT操作系统6.828补充知识：进程、页、地址空间..."
keywords: ""
date:   2017-02-27 14:40:22 +0800
---

### Process
- Page tables（xv6用的）是由硬件去实现的，给予给一个进程提供自己的地址空间。它的作用就是将一个虚拟地址（一段x86指令操作的地址）转变为物理地址（处理器芯片将该地址送到主内存）。

- 而一个Process的最重要的内核状态就是它的page table、内核栈和运行状态。
- 每个进程都有两个栈：用户栈和系统栈。
- 当进程进行一次system call时，处理器切换到内核栈，提高硬件的特权级别并且开始执行内核指令来实现System Call。call结束时，内核返回到用户空间：硬件降低它的设备特权，重心切换大用户栈，接着继续执行用户指令。


###### 那么第一个进程是怎样创建的呢？

**首先要准备好第一个进程的状态**

userinit (2502)  第一个进程才调用它
   ---> call allocproc (2456) 它被userinit调用，并且每个新进程的创建都需要调用它。它的作用就是在proc table中分配一个slot，然后初始化其内核线程需要执行的进程状态。整个逻辑大概是：先找到一个```UNUSED```的slot，然后设置为```EMBRYO```，并且给它一个独一无二的pid，接着给该进程的内核线程分配一个内核栈。


 第一个进程想执行一段小程序，而要执行该程序，进程就必须要有物理内存来储存它，然后将其复制到内存执行。这就需要用到page table。userinit中调用的setupkvm (1837)就是作的这个工作。

一旦进程被初始化，就被标记为```RUNNABLE```。


**运行第一个进程**

在第一个进程状态准备好之后，就可以开始运行它了。在main调用了userinit之后，就调用mpmain函数调用scheduler来运行进程。Scheduler就寻找设置为RUNNABLE的状态的进程，此时只有一个initproc。然后在执行过程中改变页表。Scheduler之后就将进程的状态设置为RUNNING，并且调用swtch（2958）执行上下文切换到目标进程的内核线程。并在ret指令下结束上下位切断。此时，处理器就在进程p内核栈的上RUNNING。



### Page tables

vx6主要用page tables来多路映射地址空间并保护内存。接着具体解释一下x86硬件提供的page tables以及xv6是怎样使用它的。

**paging 硬件**

不管是user也好还是kernel也好，都是用x86指令来操纵虚拟地址。而机器的RAM或者物理内存其实就是物理地址的索引。page table硬件通过映射每个虚拟地址到物理地址来连接这两个类型的地址。



一个x86的page table有2^20 (1,048,576)这一串page table 入口（PTES）。每个PTE包括来一个20位的物理页号（PPN）和一些标记位。这个paging硬件通过利用虚拟地址的高20位作为page table找到一个PTE的索引来转换虚拟地址，然后用找到的PTE中的PPN来替换虚拟地址的高20位，同时将虚拟地址中未改变的低12位转换到物理地址上。因此，一个page table就能够使操作系统可以进行虚拟-物理地址的转换，而这对齐的4096（2^12）bytes这一块就叫做page。

现在根据下图来解释page table：

![x86 page table hardware](https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/lab2/x86_pagig.png?raw=true)

虚拟地址到物理地址到转换需要经过两步，页表在物理内存中的存储方式是以两层树的方式进行的。树的根就是一个占据4096bytes的页目录，包含了1024条指向页表的32位的PTEs。首先，paging硬件利用虚拟地址的最高10位来选择一个页目录的入口地址，若该地址是存在的，那么进行第二步，也就是利用虚拟地址接下来的10位来选择页目录所指向的页表。当然，如果这两步中存在一步发生没有选择的地址，那么会产生转换失败。每个PTE包含PNN的同时，其FLAG存储着该地址的信息。


### 进程地址空间

当在main中调用函数kvmalloc时，操作系统就切换到了另一个页表，这是由于内核对进程空间的描述有更加周密的方案。

每一个进程又一个独立的页表，并且xv6要求页表硬件：当进程进程切换时，页表也要进行切换。

Xv6用页表来给每一个进程一个地址空间。如图，地址空间包括从地址0开始的用户内存，紧接着时指令、全局变量，然后时栈，最后是堆。
![space](https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/lab2/space.png?raw=true)

再看下图，这是一个进程的虚拟地址空间以及物理地址空间的布局。注意一个机器有大于2G的物理内存，xv6只能利用KERNBASE到0xFE00000之间的内存。一个进程的用户内存从地址0开始，可以增长到KERNBASE，允许一个进程地址达到2GB内存（包括内核）。文件memlayout.h (0200)声明了xv6内存布局的常量和转换虚拟地址到物理地址到宏。

![Layout of the virtual address space of a process and the layout of the physical address space](https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/lab2/process_space.png?raw=true)

当一个进程需要向xv6申请更多的内存时，xv6首先找空闲的页表来提供储存，然后将PTEs加入到指向到新的物理地址的进程页表中。设置PTE的FLAG，大多数进程不会利用到整个用户内存地址空间，所以xv6就清理那些空闲的PTEs的PTE_P。不同的进程的页表将用户地址转换为不同的物理内存的页，以至于每个进程都有私有的用户内存。

xv6还包括对所有内核进程页表的映射。它将虚拟地址KERNBASE:KERNBASE+PHYSTOP映射到 0:PHYSTOP，原因一是内核能使用它自己的指令和数据。另一个理由是，内核有时需要能过对一个物理内存给定的页进行写操作。当然，这样做对缺陷就是xv6不能利用大于2GB的物理内存。一些设备也是可以从虚拟地址映射到物理地址。

这样使得所有的页表都包括了用户内存和整个内核，这在系统调用和中断时，切换用户代码和内核代码的时候就非常方便了：这样的切换不需要页表的切换。内核并没有它自己的页表，它总是借助进程的页表。

总的来说就是，xv6保证每个进程可以使用它自己的内存，并且视自己的内存为从地址0开始的毗邻的虚拟地址。

```
code：
main 调用 kvmalloc (1857)
setupkvm (1837)
mappages (1779)  kmap(1828)
walkpgdir (1754)
```















