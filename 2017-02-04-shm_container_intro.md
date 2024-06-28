---
title: shm_container发布1.0.0-beta
date: 2017-02-04 22:00:00 +0800
categories: [open-source]
tags: [shm, C++]
math: false
---
## 背景 - 共享内存模式

要理解这个组件的作用，得从共享内存(SHM)说起。共享内存通常被理解为一种IPC（进程间通信）机制，实际上它的意义远多于此。我之前所工作过的鹅厂团队或许是最早把它玩转，并作为所有服务器软件的开发基石来使用的，可以说围绕其已有一套较完整的设计模式和方法论。这里简单列举一下这套模式的过人之处：

* 模块化与解耦，鹅厂有个方法论叫大系统小做，用现在流行词叫微服务，其实将近20年前就已经有团队用SHM来实践了
* 风险隔离与高可用，也是上一点的结果，对于C/C++服务特别有用，如果你经历过批量SEGFAULT会深有感触
* 进程重启仍能保留状态，这个在有状态服务场景非常有用，玩法很多
* 可运营性强，基本所有内存资源预分配，用量方便监控
* 极高性能，无锁无IO，比RPC方式高几个量级

客观地说，共享内存模式在十多年前的优势和必要性更大一些，因为那个时代没有RPC、没有kafka、没有Redis，性能成本更敏感。但即使今天，这套技术在高性能、高可靠的需求场景下，仍有很大的用武之地。这些年我见过不少C/C++项目，大多是线程、指针满天飞，雪崩、crash、泄漏常在，很多优化空间。

为什么对于共享内存，大家特别是小团队用得并不多呢，我想一个是缺少经验，另一个更重要的原因是，门槛确实很高。首先操作系统提供多套API，SysV-IPC、POSIX、hugeTLB选项、各种参数和坑。然后是你得自己管理内存，这比使用STL困难多了，依然有很多坑等着你。即使有一些封装库，真正能做到简单易用的也很困难。

这也是我写这个基础库shm_container的初衷，因自身有多年SHM编程经验，加上现在C++11等工具的成熟，希望重新设计一个C++库，可以更高效和可靠地使用共享内存，接近像使用STL一样容易地使用它。

## 设计 - 小而美且壮

下面简单介绍下这个库的架构，如下图：

![](/res/201702-shm_container_intro/shm_container_arch.png)

基本分成两层，上面一层是容器层，提供了使用共享内存进行存储、传递数据的高级接口；底下一层是共享内存分配器的抽象和实现层，可以让不同的容器方便地选择使用任何一种共享内存机制。分别具体一下，先看容器：


|    Container      |         Description                               |
|-------------------|---------------------------------------------------|
|    ShmArray       |  存储固定大小的结构体或结构体数组                      |
|    ShmQueue       |  单写单读无锁FIFO队列，支持ZeroCopy方式读写            |
|    ShmSyncBuf     |  单写多读的无锁广播队列，多用于数据分发或同步            |
|    ShmHashTable   |  高性能固定结点大小多阶Hash表，这里有[算法介绍][ht]      |
|    ShmLinkTable   |  基于块链表的变长内存分配器                           |
|    ShmHashMap     |  值可变长的HashMap，它基于ShmHashTable和ShmLinkTable |

共享内存分配器有：


|    Allocator      |         Description                               |
|-------------------|---------------------------------------------------|
|    SVIPC          |  基于最传统的SysV-IPC的共享内存分配器                  |
|    SVIPC_HugeTLB  |  使用HugeTLB大页的共享内存分配器，需通过SysV-IPC接口    |
|    POSIX          |  基于POSIX共享内存接口的分配器，一大优点是字符串名字     |
|    ANON           |  基于共享匿名页的分配器，可以父子进程间数据共享          |
|    HEAP           |  基于进程私有内存的分配器(并不是共享内存了)             |

还有的一些优点：

* 精心设计的接口，简单程度接近于STL
* 充分的防御性编程代码和异常检查代码
* 充分的单元测试，目前约有200+单测用例
* 小巧，基础库代码全.h，且无任何第三方依赖
* 充分的文档注释，也提供在线文档
* 代码完全符合google C++ code style

## 性能 - 目标极致

从设计到实现，高性能始终会作为主要的考虑。所有实现均为无锁实现。对基础库的大部分接口，都会有[gtestx](https://github.com/mikewei/gtestx)性能单测。直接贴下数据：

|    Container      |     Operation      |      OPS       |
|-------------------|--------------------|----------------|
|    ShmQueue       |       写入          |   1200万/s     |
|    ShmSyncBuf     |       读出          |   1500万/s     |
|    ShmHashTable   |       查询          |   1500万/s     |
|    ShmLinkTable   |       读取          |   2000万/s     |
|    ShmHashMap     |       查询          |   500万/s (*)  |

(*) 此接口目前仅提供了std::string版，性能部分受限于动态内存分配


## 资源

最新代码可在github上获得：[https://github.com/mikewei/shm_container](https://github.com/mikewei/shm_container)

在线文档可在此查看: [SHM-Container Documentation](https://mikewei.github.io/doc/shm_container)


[ht]: https://github.com/mikewei/blogs/blob/master/201512-high_performance_hash_table.md
