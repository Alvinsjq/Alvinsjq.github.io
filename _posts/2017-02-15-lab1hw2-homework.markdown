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


**格式化打印到控制台**

事实上，对于C语言到printf(),我们依旧需要在操作系统中去实现这样的I/O。

通读kern/printf.c, lib/printfmt.c 和 kern/console.c文件，确保能理解它们的关系。
在后面的实验中会知道为什么printfmt.c文件单独在lib文件夹下。


#### 代码分析


在print.c文件就像按照注释说的一样：是内核输出到控制台cprintf的简单的实现，该实现基于printfmt和内核控制台的cputchar。因此这个文件依赖于console.c和printfmt.c这两个文件。

*那么就先看一下console.c的cputchar函数：*

```c
//高级别控制台I/O，readline和cprintf用到的
void
cputchar(int c)
{
    cons_putc(c); 
}
```

调用的cons_putc函数为：

```c
//输出一个字符到控制台
static void
cons_putc(int c)
{
    serial_putc(c);
    lpt_putc(c);
    cga_putc(c);
}
```

首先是serial_putc，可以看下面的代码，发现这是在控制端口0x3F8，inb后面的值计算出来为0x3FD，out后面的值计算出来是0x3F8。总得来说就是通过inb指令判断缓冲是否为空，而outb就是将要发送的数据发送到到端口0x3F8，这样该程序就将一个字符传给端口0x3F8，具体该端口的相关概念可以移步[这里](http://bochs.sourceforge.net/techspec/PORTS.LST):

```c
#define COM1        0x3F8
#define COM_TX      0   // Out: Transmit buffer (DLAB=0)
#define COM_LSR     5   // In:  Line Status Register
#define   COM_LSR_TXRDY 0x20    //   Transmit buffer avail

static void
serial_putc(int c)
{
    int i;

    for (i = 0;
         !(inb(COM1 + COM_LSR) & COM_LSR_TXRDY) && i < 12800;
         i++)
        delay();

    outb(COM1 + COM_TX, c);
}
```

 接着是lpt_putc和cga_putc了，它们的作用主要是将字符输出给并行端口，最后将它们输出到显示屏上去。


*然后就是printfmt.c的代码了，重点关注vprintfmt函数*

printfmt.c文件被许多printf风格的函数所调用，例如rintf, sprintf, fprintf等，并且这个代码可以用时给内核与用户程序所使用。
那么现在就看下vprintfmt函数：

```c
// 格式化和输出一个字符串的主要函数。
void
vprintfmt(void (*putch)(int, void*), void *putdat, const char *fmt, va_list ap)
{
    register const char *p;
    register int ch, err;
    unsigned long long num;
    int base, lflag, width, precision, altflag;
    char padc;

    while (1) {
        while ((ch = *(unsigned char *) fmt++) != '%') {
            if (ch == '\0')
                return;
            putch(ch, putdat);
        }

        // Process a %-escape sequence
        padc = ' ';
        width = -1;
        precision = -1;
        lflag = 0;
        altflag = 0;
    reswitch:
        switch (ch = *(unsigned char *) fmt++) {

        // flag to pad on the right
        case '-':
            padc = '-';
            goto reswitch;

        // flag to pad with 0's instead of spaces
        case '0':
            padc = '0';
            goto reswitch;

        // width field
        case '1':
        case '2':
        case '3':
        case '4':
        case '5':
        case '6':
        case '7':
        case '8':
        case '9':
            for (precision = 0; ; ++fmt) {
                precision = precision * 10 + ch - '0';
                ch = *fmt;
                if (ch < '0' || ch > '9')
                    break;
            }
            goto process_precision;

        case '*':
            precision = va_arg(ap, int);
            goto process_precision;

        case '.':
            if (width < 0)
                width = 0;
            goto reswitch;

        case '#':
            altflag = 1;
            goto reswitch;

        process_precision:
            if (width < 0)
                width = precision, precision = -1;
            goto reswitch;

        // long flag (doubled for long long)
        case 'l':
            lflag++;
            goto reswitch;

        // character
        case 'c':
            putch(va_arg(ap, int), putdat);
            break;

        // error message
        case 'e':
            err = va_arg(ap, int);
            if (err < 0)
                err = -err;
            if (err >= MAXERROR || (p = error_string[err]) == NULL)
                printfmt(putch, putdat, "error %d", err);
            else
                printfmt(putch, putdat, "%s", p);
            break;

        // string
        case 's':
            if ((p = va_arg(ap, char *)) == NULL)
                p = "(null)";
            if (width > 0 && padc != '-')
                for (width -= strnlen(p, precision); width > 0; width--)
                    putch(padc, putdat);
            for (; (ch = *p++) != '\0' && (precision < 0 || --precision >= 0); width--)
                if (altflag && (ch < ' ' || ch > '~'))
                    putch('?', putdat);
                else
                    putch(ch, putdat);
            for (; width > 0; width--)
                putch(' ', putdat);
            break;

        // (signed) decimal
        case 'd':
            num = getint(&ap, lflag);
            if ((long long) num < 0) {
                putch('-', putdat);
                num = -(long long) num;
            }
            base = 10;
            goto number;

        // unsigned decimal
        case 'u':
            num = getuint(&ap, lflag);
            base = 10;
            goto number;

        // (unsigned) octal
        case 'o':
            // Replace this with your code.
            putch('X', putdat);
            putch('X', putdat);
            putch('X', putdat);
            break;

        // pointer
        case 'p':
            putch('0', putdat);
            putch('x', putdat);
            num = (unsigned long long)
                (uintptr_t) va_arg(ap, void *);
            base = 16;
            goto number;

        // (unsigned) hexadecimal
        case 'x':
            num = getuint(&ap, lflag);
            base = 16;
        number:
            printnum(putch, putdat, num, base, width, padc);
            break;

        // escaped '%' character
        case '%':
            putch(ch, putdat);
            break;

        // unrecognized escape sequence - just print it literally
        default:
            putch('%', putdat);
            for (fmt--; fmt[-1] != '%'; fmt--)
                /* do nothing */;
            break;
        }
    }
}
```

上面这个函数，也就是```vprintfmt(void (*putch)(int, void*), void *putdat, const char *fmt, va_list ap)```有四个参数：

- void (\*putch)(int, void\*)
这是在printf.c文件中的函数，因此这里是个函数指针，如下：

```c
static void
putch(int ch, int *cnt)
{
    cputchar(ch);
    *cnt++;
}
```

int代表是想要输出在控制台上的字符，void* 是要输出的地址。


- void \*putdat
这表示输入的字符存放的内存地址的指针；


- const char \*fmt
fmt就是格式化的字符串，也就是printf("string sentence",..)中的引号里的内容。

- va_list ap

当在一个fmt中出现多个参数时，就需要ap来完成输出。


分析这个函数，其中在while(1)中，下面的循环代码首先输出%之前的字符：

```c
    while ((ch = *(unsigned char *) fmt++) != '%') {
            if (ch == '\0')
                return;
            putch(ch, putdat);
        }
```

之后的代码就处理格式化，例如%d、%c、%u等。




那么最后看一下printf.c的代码，重点看cprintf函数：

```c
int
cprintf(const char *fmt, ...)
{
    va_list ap;
    int cnt;

    va_start(ap, fmt);
    cnt = vcprintf(fmt, ap);
    va_end(ap);

    return cnt;
}
```

这个就是类似于printf的输出函数，关于va_start、va_end、va_list可以看[这里](http://blog.csdn.net/edonlii/article/details/8497704)。


有了这些分析就可以做练习和回答问题了。





>练习8:我们省略了一小段代码--如果是要用符号“%o”输出八进制数的话，这段代码是必须的。找到并填补这段代码。

可以参照16进制的格式化方法，对8进制的代码进行修改，将其中的代码：

```c
            // (unsigned) hexadecimal
case 'x':
            num = getuint(&ap, lflag);
            base = 16;
        number:
            printnum(putch, putdat, num, base, width, padc);
            break;

case 'o':
            // Replace this with your code.
            putch('X', putdat);
            putch('X', putdat);
            putch('X', putdat);
            break;
```

替换为：

```c
case 'o':
            // After replace the code.
            num = getuint(&ap, lflag);
            base = 8;
            goto number;
            break;
```

回答下面的问题：

1. 解释printf.c 和 console.c之间的接口，特别地，console.c 文件输出的是什么函数？这个函数是怎样被printf.c利用的？
A: 根据代码，很容易看出console.c中的cputchar函数被printf.c中的putch函数所调用。

2. 解释下面来自console.c的代码

```c
1      if (crt_pos >= CRT_SIZE) {
2              int i;
3              memmove(crt_buf, crt_buf + CRT_COLS, (CRT_SIZE - CRT_COLS) *               sizeof(uint16_t));
4              for (i = CRT_SIZE - CRT_COLS; i < CRT_SIZE; i++)
5                      crt_buf[i] = 0x0700 | ' ';
6              crt_pos -= CRT_COLS;
7      }
```

A：这段代码是在cga_putc(int c)函数中的，属于向显示器输出字符。具体的就是将目前所显示的字符整体向上移一行以便显示出下一行的字符内容。

crt_buf:字符数组缓冲区，存放的自然是要显示到控制台上的字符；
crt_pos:表示当前最后一个字符显示在屏幕上的位置；

早期计算机控制台最多显示25行X80列的字符，因此要想在控制台上显示字符，就得指定要显示的位置，以及将显示的字符告诉cga。
CRT_SIZE=80*25,那么这个if语句就是当下一个要显示的字符超过屏幕所能显示的情况下，也就是屏幕所能显示的字符满时，则需要将整体向上移动一行，for循环干的就是将接下来的一行编程空格行，最后在将crt_pos的值修改一下。


3. 下面的问题也行需要查看课时2的讲义，讲义中介绍了x86中GCC的调用习惯。

一步一步追踪以下代码的执行：

```c
int x = 1, y = 3, z = 4;
cprintf("x %d, y %x, z %d\n", x, y, z);
```

*在cprint（）的调用中，fmt指向什么？ap指向什么？*

有上面对该函数的分析容易知道，fmt显示的就是```x %d, y %x, z %d\n```，而ap是va_list类型，因此是可变参数个数类型，显然ap指向的是所有输出参数的集合。

*按照执行的顺序列出所有对cons_putc, va_arg，和vcprintf的调用。对于cons_putc，列出它所有的输入参数。对于va_arg列出ap在执行完这个函数后的和执行之前的变化。对于vcprintf列出它的两个输入参数的值。*

整个过程大致为vprintfmt函数分析fmt字符串，当x %d时，它就会选择如下处理方法：

```c
// (signed) decimal
        case 'd':
            num = getint(&ap, lflag);
            if ((long long) num < 0) {
                putch('-', putdat);
                num = -(long long) num;
            }
            base = 10;
            goto number;
```

其中会接触到一个getint函数，它主要就是为了从ap列表中得到相应类型的参数：

```c
static long long
getint(va_list *ap, int lflag)
{
    if (lflag >= 2)
        return va_arg(*ap, long long);
    else if (lflag)
        return va_arg(*ap, long);
    else
        return va_arg(*ap, int);
}
```

这样就调用到va_arg方法，在x %d下会返回第三个return，调用ap中包括的三个参数的内容1，3，4。调用一段时间ap中还有y，z的内容。这样case d下的num就是1。之后再挑战到number：

```c
number:
            printnum(putch, putdat, num, base, width, padc);
            break;
```

查看代码，cons_putc的参数就是要输出到控制台上的字符c；
vcprintf的参数时fmt和ap；
可以在monitor.c文件中添加这段代码，实现在控制台上显示字符。

```c
void
monitor(struct Trapframe *tf)
{
    char *buf;

    cprintf("Welcome to the JOS kernel monitor!\n");
    cprintf("Type 'help' for a list of commands.\n");

    int x = 1, y = 3, z = 4;                        ==> added
    cprintf("x %d, y %x, z %d\n", x, y, z);         ==> output

    while (1) {
        buf = readline("K> ");
        if (buf != NULL)
            if (runcmd(buf, tf) < 0)
                break;
    }
}
```

4.  运行一下代码：

```c
    unsigned int i = 0x00646c72;
    cprintf("H%x Wo%s", 57616, &i);
```

输出的是什么？一步步解释一下为什么有这样的输出？

通过修改monitor.c文件可以直接在控制台上观察整个结果：

```c
void
monitor(struct Trapframe *tf)
{
    char *buf;

    cprintf("Welcome to the JOS kernel monitor!\n");
    cprintf("Type 'help' for a list of commands.\n");

    unsigned int i = 0x00646c72;                        ==> added
    cprintf("H%x Wo%s", 57616, &i);                     ==> output

    while (1) {
        buf = readline("K> ");
        if (buf != NULL)
            if (runcmd(buf, tf) < 0)
                break;
    }
}

```

结果是输出```He110 World ```。

分析：
57616的十六进制数为0xe110；
i的话就视为字符串0x00646c72，而%s就能输出r=0x72，l=0x6c，d=0x64。

5. 看下面的代码，在'y='后面会输出什么？为什么会这样？

```c
cprintf("x=%d y=%d", 3);
```

只能得到 ```x=3，y=另一个值```。

6. 我们说GCC改变了它的调用规则以便它在栈上push声明序列下的argument，以至于最后一个argument最后push。你怎样改变cprintf或者它的接口，以便它依旧可能有变量的argument来通过它？

>[Push an integer after the last argument indicating the number of arguments.](https://github.com/Clann24/jos/tree/master/lab1)












