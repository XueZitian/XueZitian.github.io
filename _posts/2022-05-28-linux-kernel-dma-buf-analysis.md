---
layout: post
title: Linux内核DMA-BUF分析
---

DMA-BUF子系统提供了一个共享buffer的框架，用于多个设备驱动和子系统间的访问，以及访问间的同步。

一共包含了三个组件：dma_buf、dma_fence和dma_resv。

### dma_buf

使用一个sg_table来描述buffer，并包含一组管理操作：“struct dma_buf_ops”，可以基于这组操作获取设备地址(map)，映射到内核态地址空间或者IO空间(vmap)，映射到用户态地址空间(mmap)等。并且，可以导出为一个文件描述符，在用户态传递。

比如，cpu和gpu间共享一块内存，这块内存可以在cpu内存中，也可以在显存中。当buffer是在显存中时，cpu的访问路径可以是PCIe bar空间的MMIO，如果是用户态用户则调用mmap映射到用户态地址空间，如果是内核态用户则调用vmap映射到内核地址空间。访问路径还可以是DMA，这个时候可以调用map获取设备物理地址，如果访问路径经过IOMMU，还可以调用vmap映射到IO地址空间。

dma_buf建立了一个exporter和importer模型：
1. exporter
    创建buffer，并实现buffer管理的操作：“struct dma_buf_ops”；管理buffer后端内存分配的细节，定义真实的后端内存设备是什么，各种地址空间的映射关系，buffer scatterlist的维护等。
2. importer
    使用共享的buffer，无需关系buffer如何被分配，放在哪里。只需要根据自己的需求，借助对应的操作函数，将memory映射到自己的地址空间，然后访问它。

### dma_fence

提供了一个同步机制。

比如，cpu发送了一条命令给gpu，cpu需要获知gpu已经执行完这条命令，那么就可以借助dma_fence：cpu发送命令时绑定一个dma_fence，gpu执行完命令后，向fence发射一个信号；而cpu在发送命令后，或者阻塞等待fence发射信号，或者向fence注册一个钩子函数，这样当fence发射信号的时候，就可以唤醒cpu进程，或者调用注册的钩子函数。

### dma_resv

提供了一个buffer共享和独占的机制，比如可以同时有多个读对象，但同一时间只能有一个写对象。dma_buf配合共享fence和独占fence实现该机制，并使用ww_mutex、seqcount_mutex和RCU保护了多个访问者间竞争。

比如，gpu渲染输出的framebuffer需要交给display显示，这种场景下就可以借助dma_resv来实现同步。首先，gpu创建了一个dma_buf作为framebuffer，并向dma_resv添加了一个独占的dma_fence，紧接着gpu将该dma_buf共享给了display，但是这个时候gpu可能并没有渲染完成，所以display借助这个独占的dma_fence等待dma_buf就绪；当gpu渲染完成时，会发送一个信号给dma_fence，此时就会唤醒display，去显示这个framebuffer。这里忽略了窗口系统和多gpu渲染的情况，当有窗口系统和多gpu存在时，可能一次gpu渲染的输出仅仅是显示屏幕上的一个窗口，那么这个时候就会存在多个dma_buf的共享和同步，情况就会很复杂，所以dma_resv提供了几组锁来保证同步和防止死锁。

## Reference
1. linux_kernel/Documentation/driver-api/dma-buf.rst
