---
layout: post
title: Linux内核ww_mutex分析
---

linux内核的DMA-BUF组件为了实现buffer的共享，使用了两把锁：ww_mutex和seqcount_mutex。其中，ww_mutex可以防止多个buffer共享时出现死锁；seqcount_mutex可以实现无锁读，并且写可以抢占读。

### Mutex

ww_mutex是基于mutex锁实现的，首先分析下Mutex的实现。

Mutex是linux的一个基础锁原语，可以使用该互斥锁实现对共享内存的串行访问。Mutex是一个睡眠锁，与二值信号量类似，相比二值信号量，mutex的使用接口更加简单，但是mutex占用的内存比较大。

Mutex使用了一个原子变量(->owner)，其中包含了一个指向任务的指针(struct task_struct *)，这个指针对齐L1缓存Cache Line大小。如果这个指针为空，说明当前锁是空闲的，否则说明已经有任务获取了这把锁，指针指向的即拥有该锁的任务。

获取mutex的处理，一共包含三个路径，根据锁的不同状态选择不同的路径，从而使锁具有更高的效率：
1. fastpath：使用cmpxchg()原子的测试owner的状态，如果锁空闲，原子的将当前任务指针赋值给owner，从而快速的获取锁；否则跳转到midpath。
2. midpath： 如果锁的持有者(任务)正在运行，并且当前并没有更高优先级的就绪任务，那么当前希望获取锁的任务可以先自旋的等待，而不是直接睡眠。这样做是因为正在运行的任务可能很快就会释放锁（通常在代码设计的，会遵循持有锁的时间尽量短的原则），如果在下次OS重新调度任务时，锁仍然没有被之前任务释放，那么就会退出自旋，跳转到slowpath，这部分逻辑使用了MCS lock来实现。
3. slowpath: 当跳转到这个路径，说明前两个路径都没能成功，这个时候就是能把当前任务添加到锁的wait-queue，然后睡眠当前任务，直到锁被释放再唤醒该任务。

Mutex具有以下规则：
1. 同一时间内只能有一个任务持有锁；
2. 只有锁的owner(即持有锁的任务)可以释放该锁；
3. 不能递归的获取和释放锁；
4. 一个任务不可以在持有锁期间退出；
5. 必须使用提供的API来初始化一把锁；
6. 被持有的锁不能被重新初始化；
7. 被持有的锁的内存不能被释放；
8. 锁不能用于软/硬中断上下文中。

### ww_mutex

GPU驱动会涉及到很多buffer的操作，这些buffer可能来自显存(VRAM)或者CPU内存，它们在不同的上下文或进程之间共享。可以使用dma-buf实现这些buffer在不同设备间共享，为了实现共享机制，必然会引入锁，当一个驱动希望获取buffer的时候，需要首先获取到锁，当buffer数量增多的时候，我们如果没有机制保证所有的进程以相同的顺序访问这些锁，就会有死锁的风险。正是为了防止出现死锁，设计出ww_mutex。它的原理是将若干buffer划归为一个class，每个buffer都包含一个锁，如果一个进程希望获取buffer，首先这个进程申请一个ticket，然后该进程尝试获取buffer对应的锁，如果锁已经被另一个进程持有，那么两个进程比较tickets，小ticket的进程获得锁，大ticket的进程主动释放已经持有的锁，这样具有最小ticket的进程就可以获得这个类中的任意buffers，而大ticket的进程则要不断的重试，最终等小ticket的进程使用完buffer，它也会获取到buffer。通过给使用者分配不同的tickets，从而保证了多个进程顺序访问多个buffer，避免了死锁发生，这样的代价是部分进程需要主动放弃自己已经持有的锁，相当于通过伤害自己来避免整个系统死锁，所以称作wound-wait。

ww_mutex有两种死锁处理方法：
首先，我们定义：一个尝试获取锁的上下文或进程称作一个transaction。
1. wait-die:  
    如果持有锁的transaction更年轻，那么尝试获取锁的transaction等待；  
    如果持有锁的transaction更老，那么尝试获取锁的transaction主动回退，即释放已经持有的锁，然后进入睡眠状态，等待锁空闲。
2. wound-wait:  
    如果持有锁的transaction更年轻，那么尝试获取锁的transaction伤害持有锁的transaction，请求持有锁的transaction释放锁，并停止transaction；  
    如果持有锁的transaction更老，那么尝试获取锁的transaction等待。

相较wait-die，wound-wait方法有更少的回退发生，但是恢复回退时，wound-wait需要做更多的工作。Wound-wait是一个抢占算法(一个transaction被另一个transaction伤害)，它需要一个可靠的方式去提取伤害条件和抢占正在运行的transaction。需要注意的是，这里的抢占和进程抢占不同。一次transaction的抢占是一个transaction被伤害后并直接结束。
结束（die）

获取锁有两个函数：
1. ww_mutex_lock():
    正常的锁获取路径，如果检测到死锁风险，该个路径会直接退出，而不是block进程，返回值为-EDEADLK。
2. ww_mutex_lock_slow():
    当ww_mutex_lock提前退出时，说明该进程竞争锁失败，首先需要主动释放已经持有的锁，然后使用ww_mutex_lock_slow()重新等待锁空闲，获取到这把锁后，再逐个获取之前主动释放的锁。这条路径不会提前退出。

### seqcount_mutex

seqcount_mutex比较容易理解，它通过一个计数值实现了无锁读，写可以抢占读。具体的原理是：
1. 读操作之前，先获取计数值，如果计数值是偶数，说明目前没有写操作，可以开始读；
2. 读取结束后，再次获取计数值，如果不等于读操作之前获取的计数值，说明读操作期间发生了写操作，读到的数据可能不一致，需要重新读取；
3. 写操作之前，将计数值加1，这时计数值变为奇数，提示读操作，目前有一笔写操作正在进行，读需要等待；写操作完成，再加1，这时计数值又变回偶数；
4. 由于可能会有多个写操作同时发生，为了保证写操作的原子性，需要为写操作增加一把锁，这把锁可以是spinlock，raw_spinlock，rwlock，mutex，ww_mutex。

seqcount_mutex比较适合用在写的次数远小于读的场景。

## Reference
1. linux_kernel/Documentation/locking/mutex-design.rst
2. linux_kernel/Documentation/locking/ww-mutex-design.rst
3. linux_kernel/Documentation/locking/seqlock.rst
