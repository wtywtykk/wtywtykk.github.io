---
layout: post_layout
title: 调试器设置导致的STM32栈溢出
time: 2019年12月15日 星期日
location: 成都
pulished: true
excerpt_separator: "正好"
---

在写一个无线控制的程序，环境是STM32和VisualGDB

调试时发现开始调试后第一次执行会自动重启，重启之后能够正常工作。

进一步发现总在一个变量范围检查的assert上出错，但是这个变量在程序刚启动时候又是对的。所以考虑是有溢出。

先下内存断点看一下，但是总是断在奇怪的函数调用上面，断点附近也没有指针操作。因为不清楚GDB对内存断点支持到底怎么样，所以怀疑内存断点的功能有问题，就没有继续研究。

正好程序里面对DMX和无线部分有大改，从出问题的嫌疑上看，自然就怀疑是这部分的问题。

把无线部分关掉，没有效果，DMX部分代码往下删，发现删掉读取连接状态和收数据包的代码之后就正常了，保留任何一个都会触发问题。

但是读取连接状态的代码很短，而且里面只有读取的代码，不会写入，又可以排除嫌疑。

结合前面内存断点总是在函数调用时候断下来这一点，怀疑是栈溢出。看了下SP指针，发现果然在0x20005exx附近，恰好落在出错的数组的位置，原因找到了。

可是我明明给栈分配了32k的空间，讲道理很难用完，于是开始下断点看哪个函数占了这么多栈空间。结果发现，在进main之前SP指针就在0x20006000了,而我在lds里面指定的是0x20010000，明显不正常。

因为栈指针正常情况是由bootloader写入的，怀疑bootloader有问题。跳转到App的代码差不多下面的样子：

```c
void __attribute__((naked, noreturn)) Bootloader_RunApp_stub(uint32_t AppAddr)
{
	UNUSED(AppAddr);
	asm("ldr sp,[r0]");
	asm("ldr pc,[r0,#4]");
}
```

先读取sp，然后读取初始pc。调试了一下，却发现栈指针被成功读取了？！再继续执行发现一切正常，并没有重启。

直接跑bootloader是正常的，调试启动就不正常，那也就是说可能bootloader根本没起作用。想到调试器设置有一个刷完程序自动reset的功能，平时我会勾上，但是这一次我嫌麻烦没勾。

<img src="/postimg/2019-12-14-STM32-Debugger-Caused-Stack-Overflow-Debuggr-Setting.png" width="100%" />

其实VisualGDB官方教程也说了要勾这个选项，之前会勾上也是因为这个。

[Creating Embedded Bootloader Projects with MSBuild](https://visualgdb.com/tutorials/tutorials/arm/bootloader/msbuild/)

没勾的后果是调试器还以为程序在开头，于是读取了Flash开头的栈指针（其实是Bootloader的），然后跳转到了App的起始代码，导致App的栈区域往前挪了很多，轻而易举的爆栈了。

勾上这个选项，再开始一次调试，一切正常，问题解决。

