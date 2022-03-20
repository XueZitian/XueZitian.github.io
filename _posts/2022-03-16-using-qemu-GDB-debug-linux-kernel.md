---
layout: post
---

### GDB原理

工欲善其事，必先利其器。GDB作为一个强大的调试工具，对于我们debug和analysis
代码非常有用，为了充分使用GDB的功能，有必要了解其背后的原理。

GDB功能强大，涉及诸多技术领域。我们以最常用的breakpoint为例，看看GDB如何实现
breakpoint的功能。关于GDB的详细使用方法和原理，官网提供了非常详细的文档：
[GDB User Manual][2]和[GDB Internals Manual][3]。

breakpoint的实现方法有两种：hardware和software。

hardware breakpoint一般需要soc在硬件层面提供支持，比如soc提供了一个专用的
register用来存储breakpoint address，当PC(program counter register)等于breakpoint
register的值，cpu就会触发一个exception，并报给GDB。另外，硬件层面的debug可以
实现一些更强大的功能，像data watchpoint(当一个地址发生读写操作时，可以产生
exception，很适合用来定位踩内存的问题)，非侵入式的debug (trace pc addresss or
data value等，很适合用来分析performance)。不过这些debug功能一般需要借助昂贵的
调试工具，像trace32，DS5。有兴趣的可以去详细看看arm的coresight debug框架。

software breakpoint的原理是：当我们设置一个breakpoint时，GDB将会使用一个非法或者
breakpoint专用instruction替换断点位置的instruction，它将引起一个exception，这样
当program执行到这个位置，GDB就可以借助exception去停止program。当用户想要继续执行
program时，首先GDB会恢复原来的intruction，然后继续执行。因此，被调试程序所在的内
存必须是可写的，不能存放在ROM中。顺便说一句，像linux kernel的kprobe技术的实现原
理与software breakpoint类似。

### GDB常用命令

```bash
# run program with arguments
run [arg]
break [line_num/func]
hbreak [line_num/func]
# step enter
s [count]
# step over
n [count]
# step asm enter
si
# step asm over
ni
u[nitil] line_num
# 运行完当前的循环体，在循环体外停止
u[nitil]
# 查看一个变量的值
i[nfo] var
# 查看所有全局变量
i[nfo] locals
print [express]
# 打印结构体指针的内容
p[rint] *obj
# 优化结构体显示格式
set print pretty on
display var
# backtrace stack
bt
# 程序访问某个内存单元时停止
watch
# show 源码
list
# 查看内存中的数据
examing /FMT addr
# /FMT 格式
/FMT: o(octal), x(hex), d(decimal), u(unsigned decimal), t(binary) f(float),a(address), i(instruction), c(char), s(string), b(byte), h(halfword), w(word),g(giant, 8 bytes)
# 打印结构体成员，/0 显示偏移量
ptype
# 打开代码窗口
layout src
layout asm
tui disable
```

### GDB调试linux kernel的几种方法

使用GDB调试linux kernel大概有四种方法：
1. KGDB
2. Qemu/KVM
3. JTAG
4. Core Dumps

其中，KGDB需要开启linux kernel的相关构建选项，并且远程连接到另一台机器进行调试；
JTAG需要借助硬件调试工具，像Jlink, trace32, DS5等；Core Dumps适合用来抓取一
运行时的异常现场，然后借助Crash工具分析异常现场；Qemu是借助Qemu的GDB stub在host
上调试guest的linux kernel。而我使用GDB的主要目的是分析linux kernel代码，查看复杂
的调用链，局部变量和全局变量等。显然Qemu是更加干净和简单的方案。

### 使用qemu的GDB调试linux kernel

网上有一篇非常好的教程讲解了如何使用qemu的GDB调试linux kernel，而且还配置有视频
教程，非常nice，所以请直接移步去这位大牛的文章：[Debugging the Linux kernel with
Qemu and GDB][4]。

TODO: 添加ubuntu文件系统作为调试的系统

### Reference
* [Breakpoint Handling][1]
* [Debugging kernel and modules via gdb][5]
* [GDB usage][6]

[1]: https://sourceware.org/gdb/wiki/Internals/Breakpoint%20Handling
[2]: https://sourceware.org/gdb/current/onlinedocs/gdb/
[3]: https://sourceware.org/gdb/wiki/Internals
[4]: https://pnx9.github.io/thehive/Debugging-Linux-Kernel.html
[5]: https://docs.kernel.org/dev-tools/gdb-kernel-debugging.html
[6]: https://qemu.readthedocs.io/en/latest/system/gdb.html
