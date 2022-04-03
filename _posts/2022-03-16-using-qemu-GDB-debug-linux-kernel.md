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
exception，很适合用来定位踩内存的问题)，以及非侵入式的debug(trace pc addresss or
data value等，很适合用来分析performance)。不过这些debug功能一般需要借助昂贵的
调试工具，像trace32，DS5。有兴趣的可以去详细看看arm的coresight debug框架。

software breakpoint的原理是：当我们设置一个breakpoint时，GDB将会使用一个非法或者
breakpoint专用instruction替换断点位置的instruction，它将引起一个exception，这样
当program执行到这个位置，GDB就可以借助exception去停止program。当用户想要继续执行
program时，首先GDB会恢复原来的instruction，然后继续执行。因此，被调试程序所在的
内存必须是可写的，不能存放在ROM中，像linux kernel的kprobe技术的实现原理与
software breakpoint类似。

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
i[nfo] args
# 查看所有全局变量
i[nfo] locals
# 查看栈信息
info frame
#显示挑选的栈帧
frame n
# backtrace stack
bt
# 打印变量, @表示数组，如：var@len
print [express]
# 打印结构体指针的内容
p[rint] *obj
# 优化显示格式
set print pretty on
# 动态打印变量
display var
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
# 进入和退出TUI shortkey
c-x a
# 选择键盘控制的区域
focus src
focus cmd
```

### GDB调试linux kernel的几种方法

使用GDB调试linux kernel大概有四种方法：
1. KGDB
2. Qemu/KVM
3. JTAG
4. Core Dumps

其中，KGDB需要开启linux kernel的相关构建选项，并且远程连接到另一台机器进行调试；
JTAG需要借助硬件调试工具，像Jlink, trace32, DS5等；Core Dumps适合用来抓取运行时
的异常现场，然后借助Crash工具分析异常现场；Qemu是借助Qemu的GDB stub在host上调试
guest的linux kernel。而我使用GDB的主要目的是分析linux kernel代码，查看复杂的调用
链，局部变量和全局变量等，我觉得Qemu是一种干净和简单的方案。

### 使用qemu的GDB调试linux kernel

网上有一篇非常好的教程讲解了如何使用qemu的GDB调试linux kernel，而且还配有视频
教程，非常nice，所以请直接移步到这位大牛的文章：[Debugging the Linux kernel with
Qemu and GDB][4]。

上面这篇文章只介绍到了使用initrd作为linux的根文件系统的调试环境。那么如果我们想
完整的运行ubuntu系统，只需要在linux boot的时候告诉kernel，ubuntu的根文件系统在那
个设备节点，并在这个设备节点安装好ubuntu文件系统，就可以顺利的从initrd转移到
ubuntu系统了。具体的操作步骤如下：
1. 安装ubuntu虚拟机，和正常安装ubuntu虚拟机的流程一样
```bash
qemu-img create -f qcow2 ubuntu.qcow 20G
qemu-system-x86_64 -m 4096 \
	-boot d \
	-cdrom ubuntu-20.04.4-live-server-amd64.iso \
	-drive file=ubuntu.qcow,format=qcow2
```
2. 查看安装的ubuntu虚拟机的根文件系统节点
```bash
# 运行如下命令启动虚拟机
qemu-system-x86_64 -m 4096 \
	-smp 4 \
	-drive  file=ubuntu.qcow,format=qcow2
# 在虚拟机中查看根设备节点, 即挂载在 '/' 目录的设备节点
Guest_CMD> df
Filesystem 1K-blocks Used Available Use% Mounted on
/dev/sda2         xx   xx        xx   xx          /
```
3. 使用qemu '-kernel' 参数指定的内核作为虚拟机内核，并通过 '-append root=xxx' 指
定linux的根文件系统，'xxx'就是第二步获得的根文件系统设备节点。
```bash
qemu-system-x86_64 -m 4096 \
	-smp 4 \
	-enable-kvm \
	-kernel path/to/build/arch/x86_64/boot/bzImage \
	-drive file=ubuntu.qcow,format=qcow2 \
	-s -S \
	-nographic \
	-append "console=ttyS0 root=/dev/sda2 nokaslr"
```
4. 接下来就可以在host上开启GDB调试了
```bash
 gdb vmlinux
 GDB_CMD> target remote :1234
 GDB_CMD> lk-module
```

如果你对比第二步和第三步中的linux kernel版本，会发现他们不一样，这是因为第二步中
的linux kernel来自于第一步安装的ubuntu文件系统，而第三步的linux kernel来自我们
通过 '-kernel' 参数指定的kernel。当qemu没有使用 '-kernel' 参数时，qemu完整的模拟
了x86系统启动的完整流程：BIOS -> grub -> linux kernel -> initrd -> ubuntu文件系
统；而当qemu指定'-kernel' 参数时，qemu启动虚拟机的流程变为：linux kernel ->
ubuntu文件系统(BIOS如何boot linux有待进一步分析)。另外，当我们使用'-kernel' 指定
的kernel版本和ubuntu文件系统中kernel版本不一致时，按理ubuntu文件系统的linux
module在加载的时候可能会出错，因为它和正在运行的kernel版本并不一致。
Interesting... 关于这一块涉及到的知识点很多：BOIS, grub, initrd和ubuntu文件系统
等等，这些我需要再理一理。

最近发现上面的方法，在x86架构最新的q35虚拟机上会出现无法进入ubuntu文件系统。通过
kernel log可以看到kernel在进入根文件系统之前并没有挂载磁盘设备，导致找不到设备节
点'/dev/sda2'，也就进不去ubuntu文件系统，可能是q35的设备初始化顺序变化导致，有
待进一步分析根本原因。为了能基于q35虚拟机调试ubuntu，我又找到了一种方法：启动虚
拟机的时候不通过'-kernel'参数指定linux kernel，而是在虚拟中自己构建一版kernel，
安装该kernel，并把源码和编译生成的文件拷贝到host的相同目录下，再使用第四步的方法
同样可以调试linux kenel。

### Reference
* [Breakpoint Handling][1]
* [Debugging kernel and modules via gdb][5]
* [GDB usage][6]
* [通过QEMU调试Linux内核][7]

[1]: https://sourceware.org/gdb/wiki/Internals/Breakpoint%20Handling
[2]: https://sourceware.org/gdb/current/onlinedocs/gdb/
[3]: https://sourceware.org/gdb/wiki/Internals
[4]: https://pnx9.github.io/thehive/Debugging-Linux-Kernel.html
[5]: https://docs.kernel.org/dev-tools/gdb-kernel-debugging.html
[6]: https://qemu.readthedocs.io/en/latest/system/gdb.html
[7]: https://terenceli.github.io/%E6%8A%80%E6%9C%AF/2016/06/21/gdb-linux-kernel-by-qemu
