---
title: PWN学习之The House of Force
author: Kevin。
tags:
  - Linux
  - pwn
categories:
  - ctf
date: 2022-07-26 22:52:00
img: /images/article-banner/QQ截图20220726235315.jpg
typora-root-url: ..\..\..\
---

# PWN学习之The House of Force

接触CTF那么久终于第一次鼓起勇气接触PWN了！！

pwn必备工具安装

https://github.com/Gallopsled/pwntools

https://github.com/pwndbg/pwndbg

https://github.com/david942j/one_gadget

## The top chunk

第一步当然是熟悉一下堆栈以及malloc的工作方式

实验环境下载地址 https://www.aliyundrive.com/s/sutrieb13xL

### GDB调试一个程序

进入house_of_force，使用`gdb demo`调试程序

用`b main`（break main） 给main函数下断点

`r` （run）运行程序

就会断在该画面下

![main函数断点](/images/image-20220726230954180.png)

为了防止gdb每次运行程序时都打印完整程序信息，可以使用`set context-sections code`来打印源码部分，使用`context`命令可以查看源代码

### 查看内存分配

使用`vmmap`可以查看该程序的内存分配情况，蓝色的就是堆内存分配，可以看到目前还没有数据分配到堆中

![vmmap查看内存](/images/image-20220726231948707.png)

使用`n`（next）命令执行一行代码，再次查看可以发现堆中被分配了内存

![发现堆内存](/images/image-20220726232102759.png)

### 查看堆分配

`vis`(vis_heap_chunks)查看堆内存分配情况，青色的地方就是本次malloc分配的空间

结合上面的原码发现，malloc并未分配9字节，而是分配了前八个`inline metadata`，加上后面的8*3个字节，因为这是malloc分配内存的最小单位，并以每次0x10增加，所以分配的数据 % 0x10始终为0 

`inline metadata`也可以当作是malloc本次分配的一些基本信息，如分配的大小等。由于每次都增长`0x10`个空间，所以最后一位就可以用来干别的活了，在malloc中就被设计成了`prev_inuse`标志位，如果该标志位为1，说明这个chunk的前面一个chunk正在被使用，而第一个chunk的`prev_inuse`标志位始终为1，因为在此之前没有别的chunk了

![malloc(9)](/images/image-20220726232309573.png)

连续运行多次可以发现，无论分配9个字节还是0个字节还是24个字节，每次分配的内存都是24个字节

![malloc(1|0|24)](/images/image-20220726232930659.png)

当malloc分配了25个字节时，实际上分配了0x8+0x28个空间，因为25大于24，所以必须再增加0x10个空间

![malloc(25)](/images/image-20220726233521341.png)

每次`vis`的最后都有一个`Top chunk`，这是malloc剩余可分配的内存空间，结合上面的图片可以很快发现，`Top chunk`每次减去的数量和每次分配的数量一致
