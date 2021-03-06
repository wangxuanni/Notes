---
title: 多线程基础概念
date: 2019-02-11 23:53:50
categories: 并发
description: 多线程基础概念
---
# 进程

## 进程和线程的关系与区别？

进程是资源分配的最小单元。线程是CPU调度的最小单元，一个Java程序对应着一个进程。

| 对比维度       | 多进程                                                       | 多线程                                                       | 总结     |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | -------- |
| 数据共享、同步 | 数据共享复杂，需要用IPC；数据是分开的，同步简单              | 因为共享进程数据，数据共享简单，但也是因为这个原因导致同步复杂 | 各有优势 |
| 内存、CPU      | 占用内存多，切换复杂，CPU利用率低                            | 占用内存少，切换简单，CPU利用率高                            | 线程占优 |
| 创建销毁、切换 | 创建销毁、切换复杂，速度慢                                   | 创建销毁、切换简单，速度很快                                 | 线程占优 |
| 编程、调试     | 编程简单，调试简单                                           | 编程复杂，调试复杂                                           | 进程占优 |
| 可靠性         | 进程间不会互相影响                                           | 一个线程挂掉将导致整个进程挂掉                               | 进程占优 |
| 分布式         | 适应于多核、多机分布式；如果一台机器不够，扩展到多台机器比较简单 | 适应于多核分布式                                             | 进程占优 |

## 进程间的通信方式

1. 管道pipe：管道是一种半双工的通信方式，数据只能单向流动，而且只能在具有亲缘关系的进程间使用。进程的亲缘关系通常是指父子进程关系。
2. 命名管道FIFO：有名管道也是半双工的通信方式，但是它允许无亲缘关系进程间的通信。
4. 消息队列MessageQueue：消息队列是由消息的链表，存放在内核中并由消息队列标识符标识。消息队列克服了信号传递信息少、管道只能承载无格式字节流以及缓冲区大小受限等缺点。
5. 共享存储SharedMemory：共享内存就是映射一段能被其他进程所访问的内存，这段共享内存由一个进程创建，但多个进程都可以访问。共享内存是最快的 IPC 方式，它是针对其他进程间通信方式运行效率低而专门设计的。它往往与其他通信机制，如信号量，配合使用，来实现进程间的同步和通信。
6. 信号量Semaphore：信号量是一个计数器，可以用来控制多个进程对共享资源的访问。它常作为一种锁机制，防止某进程正在访问共享资源时，其他进程也访问该资源。因此，主要作为进程间以及同一进程内不同线程之间的同步手段。
7. 套接字Socket：套解口也是一种进程间通信机制，与其他通信机制不同的是，它可用于不同进程间的进程通信。
8. 信号 ( sinal ) ： 信号是一种比较复杂的通信方式，用于通知接收进程某个事件已经发生。

# 死锁

死锁是指两个或两个以上的事务在执行过程中，因争夺锁资源而造成的一种互相等待的现象，若无外力作用两个事务都无法推进，这样就产生了死锁。

## 四个必要条件

四个条件缩写”一球夺环“

1. 互斥条件：即任何时刻，一个资源只能被一个进程使用。其他进程必须等待。
2. 请求和保持条件：即当资源请求者在请求其他的资源的同时保持对原有资源的占有且不释放。
3. 不剥夺条件：资源请求者不能强制从资源占有者手中夺取资源，资源只能由资源占有者主动释放。
4. 环路等待条件：比如A占有B在等待的资源（B等待A释放），B占有A在等待的资源（A等待B释放）。多个进程循环等待着相邻进程占用着的资源。

避免死锁可以通过破环四个必要条件之一。

## 解决死锁的方法

1. 加锁顺序保持一致。不同的加锁顺序很可能导致死锁，比如哲学家问题：A先申请筷子1在申请筷子2，而B先申请筷子2在申请筷子1，最后谁也得不到一双筷子（同时拥有筷子1和筷子2）

2. 超时，为其中一个事务设置等待时间，若超过这个阈值事务就回滚，另一个等待的事务就能得以继续执行。比如可重入锁的超时等待

3. 数据库可以及时检测出死锁，选择一个牺牲者放弃事务，即回滚undo量最小的事务。一般是用等待图（wait-for gragh）深度优先搜索的算法实现，如果图中有环路就说明存在死锁。


## 死锁、活锁与饥饿

死锁：指两个或两个以上的进程（或线程）在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将**无法推进下去**。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。

活锁：是指线程1可以使用资源，但它很礼貌，让其他线程先使用资源，线程2也可以使用资源，但它很绅士，也让其他线程先使用资源。**这样你让我，我让你，最后两个线程都无法使用资源。**

饥饿：通常因为**线程优先级使用不当**。是指如果线程T1占用了资源R，线程T2又请求封锁R，于是T2等待。T3也请求资源R，当T1释放了R上的封锁后，系统首先批准了T3的请求，T2仍然等待。然后T4又请求封锁R，当T3释放了R上的封锁之后，系统又批准了T4的请求......，**T2可能永远等待**。

# java内存模型

*注意：没有什么jvm内存模型，要么是Jvm运行时数据区，要么是Java内存模型.*

由来

计算机采用**结构化**的存储，而存储设备与处理器的运算有几个数量级的差距，所以不得不加入一层读写速度尽可能接近处理器运算速度的**高速缓存**，来作为内存和处理器之间的缓存，将运算所需要用到的数据复制到缓存中，让运算能快速进行，当运算结束后再从缓存同步回忆内存之中。这也引起了一个新的问题，**缓存一致性**。
除此之外，处理器可能会对输入代码进行乱序执行优化，因此如果一个计算任务依赖另一个计算任务的中间结果，那么其顺序并不能靠代码的先后顺序来保证。

*插一句题外话，可不可计算机全部用内存，这样就不用考虑缓存一致性了？可以，比如说谷歌公司，他的搜索引擎的数据80%存在内存里面的，才会这么快。一般的小公司如果没有钱买这么大的内存是没办法跟他做竞争的。*
那么至于Java的内存模型有什么关联呢？Java的内存模型中。主内存可以类比硬件的主内存，而每条线程的工作内存可以类比处理器高速缓存类。
这个变量是在修改后同步回主内存，在变量读取前从主内存刷新回变量值，这种以主内存作为传播媒介的方式来实现可见性的。
对一个变量执行，mx操作之前，必须把此变量同步回族内存中，一个变量在同一时刻只允许有一条进程对其进行加锁操作。

