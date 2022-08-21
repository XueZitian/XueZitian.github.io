---
layout: post
title: VFIO分析
---

## 1 VFIO概述

VFIO(Virtual Function I/O)，是Alex Williamson开发的linux kernel模块，其主要是用于虚拟化场景，将PCIe的PF和VF以pass-through的方式提供给虚拟机，从而使虚拟机直接和物理设备交互，在保证交互高效的同时，又能确保安全性，这里的安全性是指当虚拟机设备直接操作物理设备的时候，需要确保这个设备不可以访问该虚拟机以外的其它资源，不能破坏主机以及其它兄弟虚拟机的运行，比如设备如果具有DMA能力，或者说可以访问CPU内存，那么需要限制设备访问内存的区域，其只能访问属于对应虚拟机的内存区域，这需要硬件能力的辅助，这个硬件能力一般统称为IOMMU，但不同厂家有不同技术手段，intel是vt-d，arm是smmu。除了IOMMU，该模块还依赖于CPU虚拟化技术以及PCIe的SRIOV等硬件能力。比较经典的使用组合是qemu + kvm + vfio，可以实现把一个PCIe显卡pass-through给虚拟机使用。另外VFIO除了支持PCIe设备，也支持linux的platform设备，同时VFIO作为linux kernel的一个模块，其本质是借助CPU虚拟化，IOMMU和SRIOV等硬件能力，屏蔽硬件架构间的差异，抽象出一组接口，用户态程序可以使用这组接口直接访问设备。所以，除了供qemu使用，一些用户态程序也基于这组接口实现安全高效的设备访问，比如DPDK，rust-vmm。本文重点分析PCIe的PF直通给虚拟机所依赖的硬件能力，以及基于qemu的软件实现。

## 2 几个硬件能力介绍

### 2.1 IOMMU

IOMMU(Input–Output Memory Management Unit)，是一个硬件模块，其主要是用在设备访问CPU内存的场景，将设备发出的地址(IOVA)进行重映射，即IOVA转换为HPA。IOMMU和CPUMMU硬件功能类似，只是使用的场合不同，两者都是使用页表控制虚拟地址到物理地址的转换，并且都可以有多个页表上下文，对于CPU MMU，其页表上下文切换是基于线程，而IOMMU的页表上下文切换则是基于设备硬件信息，如PCIe TLP包中的request id，不同的requestid会匹配不同的页表基地址，即页表上下文，request id可以是PCIe设备的BDF(Bus,Device, Function)，所以我们通过控制页表基地址，不同的设备使用不同的页表，就可以实现对设备访问CPU内存的限制。IOMMU的使用场景主要有三个：

**1)DMA地址重映射**

我们知道CPU上运行的程序使用的都是虚拟地址，当我们使用malloc申请一块内存bufA，其虚拟地址是连续的，但其物理地址是离散的，而设备dma搬移数据是基于物理地址，那么当使用DMA在设备和CPU内存之间搬移bufA的数据时，在没有iommu的情况下，需要拆分成多笔描述符，或者使用dma的scatter-gather能力，当存在iommu时，我们就可以通过控制iommu的页表将离散的物理地址重映像为连续的设备虚拟地址，然后dma使用设备虚拟地址来搬移数据。另外，有些设备dma会受限于地址访问宽度，比如我们的CPU位宽是64bit，而dma只能发出32bit位宽的地址，在没有iommu的情况下，dma只能访问CPU低32bit区域的内存，所以如果搬移32bit以外区域的内存就需要先将数据拷贝到低32bit区域，这样做一方面会浪费CPU时间片，另一方面低32bit内存数量有限，可能申请失败；当存在iommu时，我们就可以通过控制iommu的页表将32bit以外区域的内存重映射到低32bit区域。在linux kernel中，实现上述功能的函数是：dma_map_sg()和dma_map_single()等。

**2)虚拟化场景下的物理设备Pass-Through**

这种场景下，虚拟机可以直接和物理设备交互，那么虚拟机就需要控制设备的DMA，即填写DMA描述符中的源地址和目的地址，而由于虚拟机中的物理地址GPA并不是真正的物理地址，所以DMA使用这个GPA是无法访问到正确的CPU内存的。但有了IOMMU以后，就可以使用IOMMU将GPA重映射为HPA。

**3)SVA(Shared Virtual Addressing)**

SVA技术可以实现CPU和设备使用相同的虚拟机地址去访问CPU的内存，可以省去软件将虚拟地址转换成物理地址的步骤。这种技术不同于第一种场景的用法，第一种场景CPU使用的HVA，设备使用的是IOVA，两者不是同一个虚拟地址，且在进行DMA传输之前，软件需要首先pin住CPU的page，防止页面被换出。SVA技术可以实现直接使用线程的虚拟地址进行dma传输，并且无需pin page，实现这个技术需要依赖很多的硬件能力支撑，除了IOMMU，还需要PCIe的ATS(Address Translation Services)，PRI(Page Request Interface)和PASID(Process Address Space Identifier)等能力的支持，从而实现设备在进行DMA传输时，携带进程ID用于匹配对应进程的页表，发生缺页异常时调用缺页异常处理函数，设备侧TLB没有命中时调用PRI更新TLB等。这种用法非常复杂，需要很多的硬件支撑。

x86架构的IOMMU技术，即vt-d，主要实现了两个功能：

**1)DMA Remap**

DMA Remap的地址转换支持两种模式，Legacy Mode和Scalable Mode，两者的区别在于是否使用PASID，或者说典型的使用场景下，Legacy Mode只有一级地址转换，即GPA转换为HPA，其只需要使用BDF作为request id来选择页表即可；Scalable Mode包含两级地址转换，第一级GVA转换为GPA，第二级GPA转换为HPA，其中第一级使用的request id是PASID，第二级使用的request id是BDF。这两种模式可以对应到上面的场景2和场景3。

**2)Interrupt Remap**

上面着重分析了IOMMU的DMA重映射能力，IOMMU还需要承担另一个责任：中断重映射。中断重映射可以实现中断的隔离，将设备中断分配给特定的虚拟机，另外hypervisor也可以利用中断重映射修改中断请求的属性，比如目的CPU，中断向量，分发模式等。这里的中断主要指的是MSI/MSIX这类中断，设备通过向特定范围的地址(0xFEEx_xxxx)写一笔特定数据来产生一个中断请求，这个中断会被Root-Complex识别，而IOMMU就集成在Root-Complex，中断重映射逻辑会根据写请求的地址和数据计算出一个偏移量，以及结合requst id，去寻址由hypervisor建立好的一张中断重映射表，最终基于中断向量表中的信息发送一个中断给CPU。

### 2.2 CPU虚拟化

在pass-through场景下，iommu的页表是由hypervisor控制，那么hypervisor是如何获取GPA与HPA的映射关系的呢？

我们首先来看看CPU虚拟化是如何实现的，VMX是x86架构提供的CPU虚拟化硬件能力，其主要是新增了VMX operation，包含两种模式：VMX root operation和VMX non-root operation，VMX root operation运行host和hypervisor，VMX no-root operation运行guest。这里面有一个非常关键的技术EPT(Extended Page Table)，它实现了物理内存的虚拟化，原理是在VMX non-root operation模式下，MMU的地址转换会分成两级，第一级是GVA转换为GPA，其转换页表由guest控制；第二级是GPA转换为HPA，即EPT，其转换页表由hypervisor控制。而在VMX root operation模式下，MMU地址转换只有一级，HVA转换为HPA，其页表有host控制。

显然，hepervisor在完成虚拟机物理内存虚拟化时，已经维护了GPA与HPA的映射关系，因此只需要将这个映射关系同步给IOMMU即可。

### 2.3 SRIOV

SRIOV(Single Root I/O Virtualizaton)，是PCIe规范中中定义的一个Capability，旨在提高对硬件设备的利用率，主要用于虚拟化场景下。该规范引入了一个很重要的概念：VF(Virtual Function)，每个PF可以包含多个VF，每个VF同PF一样，具有自己的配置空间，BAR Region，MSI/MSIX中断，BDF等等。这样可以比较容易的基于SRIOV规范设计出支持硬件虚拟化的设备，如一张显卡支持4个VF，4个VF分别pass-through给不同的虚拟机，这样多个虚拟机可以共享一个物理显卡，当然直接把显卡的PF pass-through给虚拟机也是可以的。VFIO的PCIe部分就是以VF和PF为粒度来封装PCIe设备，然后提供给用户态使用。

## 3 软件实现

### 3.1 Linux kernel中VFIO模块的设计

VFIO将设备暴露给用户态时，解决了一个最重要的问题，基于IOMMU限制设备的I/O访问，中断和DMA能力，确保了安全性。为了保证安全性，VFIO定义了三个概念：device，group，container。安全隔离的最小粒度是group，属于同一个group的设备必须分配给同一个虚拟机，container包含了分配给某个虚拟机的所有设备。可以通过打开"/dev/vfio/vfio"字符节点创建一个container；如果我们想添加一个设备到该container，首先需要找到该设备所属的group，然后卸载该group下所有设备的驱动，再绑定VFIO驱动，这时候就可以看到group的设备节点"/dev/vfio/$GROUP"，接下来就可以将这个group添加到container中，同一container中的所有group可以共享IOMMU页表，这样可以减少TLB miss。

### 3.2 Qmeu中VFIO模块的使用

qemu中可以基于内核提供的VFIO接口创建PCIe设备：TYPE_VFIO_PCI，比如我们添加如下命令行参数：

```bash
-device vfio-pci,host=0000:00:01.0
```

qemu基于VFIO创建PCIe虚拟设备主要做了这几件事：

**1)配置空间**

虚拟机中PCIe设备的配置空间是qemu虚拟的，qemu会申请一块host内存作为虚拟设备的配置空间，然后将物理PCIe设备的配置空间内容刷新到虚拟设备配置空间，所以可以在qemu中替换虚拟PCIe设备的配置，比如vender_id和device_id等。而虚拟机中对配置空间的读写会导致虚拟机退出到qemu，最终调用函数vfio_pci_write_config()和vfio_pci_read_config()，这两个函数会截获虚拟机的操作，并同步给VFIO和物理设备。

**2)Bar Region**

虚拟机中的PCIe Bar Region和物理PCIe Bar Region是线性映射关系，虚拟机中对Bar的MMIO读写可以直接映射到物理设备对应的Bar。原理是qemu将物理设备的Bar Region通过mmap()映射为HVA，然后将该HVA作为虚拟设备的内存后端，并调用KVM接口配置GPA到HPA的转换页表，从而借助CPU内存虚拟化技术实现虚拟机对物理设备Bar资源的直接访问，提高了I/O吞吐。Qemu中的相关函数如下：

```c
vfio_region_mmap()
|---mmap()
|---memory_region_init_ram_device_ptr()
|---memory_region_add_subregion()
    |---/* memory_listener call kvm interface for configuring EPT */
```

**3)MSIX Table**

并不是所有的Bar Region都直接映射到虚拟机中，这里的一个例外就是MSIX Table，物理设备的MSIX Table是由VFIO控制的，qemu中会申请一段host内存，作为虚拟机的MSIX Table，这段内存会覆盖物理Bar中MSIX Table那部分的直接映射（由于CPU映射的粒度是页的size，一般是4KB，所以MSIX Table区域的起始地址和size需要4KB对齐），这样虚拟机无法直接读写物理机的MSIX Table，qemu通过截获虚拟机对MSIX Table的读写操作，间接为虚拟机注册中断，并借助APIC虚拟化技术触发虚拟机中的中断。

**4)IOMMU配置**

当我们把一个VF或PF pass-through给虚拟机时，相应的需要为这个VF或PF设备配置IOMMU的页表，使其可以使用GPA来访问虚拟机所对应的物理内存，此时IOMMU负责将GPA转换成HPA，并限制设备可访问的内存区域，从而设备可以正确的访问CPU内存，并确保host及其它虚拟机的安全性。Qemu中的VFIO设备会向虚拟机的内存地址空间注册配置IOMMU的memory_listener，从而可以同步cpu内存虚拟化的映射，即GPA->HPA。Qemu中相关函数如下：

```c
// register memory_listener
vfio_connect_container()
|---memory_listener_register()
// memory_listener functions
vfio_memory_listener = {
    .region_add = vfio_listener_region_add,
    .region_del = vfio_listener_region_del,
    .log_global_start = vfio_listener_log_global_start,
    .log_global_stop = vfio_listener_log_global_stop,
    .log_sync = vfio_listener_log_sync,
}
```

## Reference
1. [An Introduction to PCI Device Assignment with VFIO][1]
2. linux_kernel/Documentation/x86/sva.rst
3. Intel Virtualization Technology for Directed I/O Architecture Specification
4. Intel 64 and IA-32 Architectures Software Developer’s Manual
5. PCI Express Base Specification Revision 4.0 Version 1.0
6. linux_kernel/Documentation/driver-api/vfio.rst

[1]: http://events17.linuxfoundation.org/sites/events/files/slides/An%20Introduction%20to%20PCI%20Device%20Assignment%20with%20VFIO%20-%20Williamson%20-%202016-08-30_0.pdf
