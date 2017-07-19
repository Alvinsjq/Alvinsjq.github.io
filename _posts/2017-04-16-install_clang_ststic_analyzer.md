---
layout: post
title:  "在ubuntu上配置Clang Static Analyzer"
tags: [clang,static analyzer]
comments: true
description: "由于这学期的系统分析与验证工具课的要求是要调研这个工具，于是记下这个作为备忘吧。
那么开始如何在linux平台上来使用基于clang的静态分析工具呢？"
keywords: ""
date:   2017-04-18 13:40:22 +0800
---
由于这学期的系统分析与验证工具课的要求是要调研这个工具，于是记下这个作为备忘吧。
那么开始如何在linux平台上来使用基于clang的静态分析工具呢？下面是根据参考资料做的一个简单的整理：

- 命令行输入```clang --version```,查看一下是否安装clang；
- 如果没有安装，执行命名```sudo apt-get install clang```进行安装；
- 创建一个memleak.c文件：
```c
#include<stdio.h>  
#include<stdlib.h>  
int main()  
{  
    int *mem;  
    mem=malloc(sizeof(int));  
    if(mem) return 1;  
    *mem=0xdeadbeaf;  
    free(mem);  
    return 0;  
} 
```
<!--more-->
- 到达mamleak.c路径下，执行 ```$ scan-build clang -c memleak.c```命令,得到类似结果：

```
alvin@AlvinUbuntu:~/clang_cases$ scan-build clang -c memleak.c
scan-build: Using '/usr/bin/clang' for static analysis
memleak.c:7:17: warning: Potential leak of memory pointed to by 'mem'
        if(mem) return 1;
                       ^
memleak.c:8:6: warning: Dereference of null pointer (loaded from variable 'mem')
        *mem=0xdeadbeaf;
         ~~~^
2 warnings generated.
scan-build: 2 bugs found.
scan-build: Run 'scan-view /tmp/scan-build-2017-04-17-144607-26548-1' to examine bug reports.
```


- 最后根据提示可以打开网页版的分析结果，即输入```scan-view /tmp/scan-build-2017-04-17-144607-26548-1```即可得到报告。

<img src="https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/additional%20screenshot/1.png?raw=true" width="500">


- 点击 **view report** 可查看详情：

<img src="https://github.com/Alvinsjq/6.828_tasks/blob/master/screemshot/additional%20screenshot/2.png?raw=true" width="360">


## 参考

- http://www.cnblogs.com/zfyouxi/p/4747792.html
- 