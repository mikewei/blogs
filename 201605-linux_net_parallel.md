### Linux网络IO并行化技术概览
[Linux系统][2016-05-21 00:30:00 +0800]

过去的十年中互联网经历了爆发式的增长，这背后有什么技术平台起了最为关键的作用，我认为是Linux，即使在云计算流行的今天，它依然是最重要的一块基石。我们或许经常听到关于什么是最好的服务器编程语言、怎样是最好的架构设计的讨论，却从未听到有人讨论什么是最好的服务器操作系统，实际上它的地位早已重要到我们习惯地把它作为一个标准而非一个选择。从2001年2.4到2003年2.6，再到2011年3.0，以至今天的4.6，Linux在性能、稳定、易用等方面持续不断的提升。作为用户很多时候我们发现问题已不在于Linux能否跟上我们的需求，而在于我们能否及时了解掌握Linux的众多功能特性而加以利用。

本文主要来聊一聊关于Linux网络IO(协议栈)的相关技术。记得大约十年前，单box最大性能可以处理上万qps，十万并发连接，而如今单box可处理数十万qps，数百万并发连接。这里除了硬件性能的提升以外，内核的优化技术也起到很大作用。另外，这些优化特性并非总是默认有效或最佳的，时有需要tuning的场景，我在近几年工作中也多次碰到服务器性能问题而通过简单tuning可以有效改善，所以理解它们挺有必要。或许你已经听说过下面这些内核支持特性：中断亲和，多队列网卡、RPS、RFS、XFS、SO\_REUSEPORT.. 对它们的介绍资料在网上也有不少，但我发现很少有从全局角度地系统性介绍，所以有了我总结此篇的动机。

首先对上面这些特性我认为可以归纳一个主线：并行化。因为网络协议栈处理，本质来说是CPU密集的计算，所以多年来各种关键优化补丁的共同思路，基本都是怎么充分利用多核的资源达到计算并行。为啥这么简单一个思路会搞出那么多概念呢，因为协议栈计算本身是一个复杂的分层的处理过程，在各个层各处理环节都有并行优化的空间，上述这些优化补丁正是在这些不同的层次的工作。下面我将按此脉络展开对这些技术点做一个介绍。

#### Linux协议栈

首先我们先回顾一下Linux协议栈的分层结构，如下图：

![overview](/res/201605-linux_net_parallel/linux_net_overview.png)

最底层是硬件网卡(NIC)，它通常通过两个内存环型队列(rx\_ring/tx\_ring)加上中断机制与操作系统进行通讯。当NIC收到数据包后，它将数据包写入rx\_ring并产生中断。CPU收到中断后OS将陷入中断处理程序中执行，这在Linux内核中叫Hard-IRQ。

在较老版内核中，网卡Hard-IRQ程序将数据包从rx\_ring中取出并放入PerCPU的一个叫backlog的队列，然后发起一个Soft-IRQ来处理backlog队列 (内核将中断处理中无需实时同步完成的工作delay到一个准实时的异步时机去执行，这个异步机制即Soft-IRQ)。较新版的内核中，通常并不使用backlog队列，而是使用叫NAPI的改进机制，区别是Hard-IRQ不再直接读取每个数据包，而是直接启动Soft-IRQ，在Soft-IRQ中通过batch poll的方式将数据包从rx\_ring中取出并处理 (可大大减少中断数)。

不管是backlog还是NAPI，它们都在Soft-IRQ上下文中执行，并把数据包提交给IP层进行处理(为简单我们都以TCP/IP协议为例)。IP层处理完分片和路由后将提交给传输层(TCP或UDP)进行处理。协议相关逻辑我们不在这里详述，最终它将此数据包放入对应的socket对象的接收队列中，并唤醒阻塞在socket上的进程。

用户态进程通过socket fd来操作内核中的socket对象，经常它会阻塞在socket相关的read/write或epoll/select的操作上。还需注意到Linux的文件机制允许多个进程通过各自的fd同时竞争操作同一个socket对象。典型地场景如，多个进程竞争accept一个listening的socket，再如，多个进程竞争读一个udp socket。

#### 中断调度

协议栈对入包的软件部分处理，总是从硬中断(Hard-IRQ)处理开始的。关于中断处理你需要了解几个事实：

* 计算机系统中有很多不同作用的中断请求，由中断号唯一标识，比如每块网卡有自己的中断号
* 对每个中断号，系统都会注册一个handler(也就我们通常说的中断处理程序)
* 在Hard-IRQ handler中(如网卡中断处理程序)通常将无需立即完成的工作(如TCP/IP协议栈处理)通过Soft-IRQ异步地执行
* Soft-IRQ顾名思义就是软件构造的类似的中断机制，它也根据用途区分不同的类型，并有对应的handler。它存在的主要意义是让中断对系统实时性的影响尽可能小

不管是Hard-IRQ还是Soft-IRQ handler，它们都是一段需要调度的执行流(就像线程一样)，那么问题来了：如何高效地调度这些执行流在多核下运行，对系统性能非常关键。下面介绍一下目前的一些调度机制：

* 对同一个中断号的Hard-IRQ handler，在全局上是串行执行的，即同时只能在一个核上执行
* 对不同的中断号的Hard-IRQ handler，可以在不同的核上并行执行
* 某个中断号在哪个核上执行，通常由系统中的I/O APIC(高级可编程中断控制器)来决定，内核提供了配置接口（也有一种称为irqbalance的动态调整工具可选）
* Hard-IRQ handler中发起的Soft-IRQ，一般在同一个core上执行

我们可以使用下面的命令观察系统中所有中断号以及它们在各core上的调度情况：

	cat /proc/interrupts

下面回到网络IO的主题上，从网卡中断的角度看协议栈处理，如下图：

![irqsched](/res/201605-linux_net_parallel/linux_net_irqsched.png)

传统的网卡每块设备有一个中断号，它由上述的调度机制为每个中断请求分配给一个唯一的core来执行。在此场景下，你会发现协议栈处理的并行化是以网卡设备为粒度的。

如果只有单块网卡，就会发现中断处理CPU消耗集中于单个核上（记得默认对应的Soft-IRQ会在同一个core上执行）；更坏的情况是，如果只有一个处理socket的应用进程，很可能你会看到所有CPU负载集中于一个core上（实际上进程调度策略会优化将唤醒的进度调度在相同的core上执行）。怎么优化？别着急下面的内容中会有很多方法。

如果有多块网卡，通常来因为利用多核并行而会获得性能的提升，如下图所示：

![irqsched](/res/201605-linux_net_parallel/linux_net_irqsched2.png)

但我们也曾发现过例外，虽然两块网卡的中断请求被调度到2个core上，但它们是[超线程](https://en.wikipedia.org/wiki/Hyper-threading)技术对同一物理core虚拟出的2个逻辑core，并不能有效地并行处理。解决方法是手工配置中断亲和（绑定中断号与具体的core），如下命令：

	echo 02 > /proc/irq/123/smp_affinity


#### 多队列网卡

如前所述，传统的单网卡默认无法充分利用多核，即使是多网卡，在数量小于核数的情况下也是一样。于是产生了在今天广泛使用的多队列网卡(Multi-Queue NIC)，在一些资料里这种技术也叫RSS(Receive-Side Scaling)。这种技术概括来说是从硬中断的层面支持了单网卡IO的并行，其工作原理如下图所示：

![multiqueue](/res/201605-linux_net_parallel/linux_net_multiqueue.png)

多队列网卡通过引入RX-Queue的机制，将输入流量水平分到多个“虚拟的网卡”也是RX-Queue，每个RX-Queue像一个独立设备一样有自己的中断号并可以独立并行地工作。RX-Queue的数量一般可以配置为与核数一致，这样可以充分利用多核资源。

值得注意的是，输入流量拆分到RX-Queue的算法是根据Hash(SrcIP, SrcPort, DstIP, DstPort)来计算的（可以试想下为什么是用hash而不是类似随机分发？对，主要是为了避免乱序）。如果出现大部分流量来自少量IP:Port的场景，多队列网卡并就爱莫能助了。

你可以使用前面提到的interrupts文件来观察多队列网卡的分发效果：

	cat /proc/interrupts

这里也[有篇文章](http://blog.csdn.net/turkeyzhou/article/details/7528182)较详细介绍了这个主题可作参考。


#### RPS

在没有多队列网卡的服务器上，比如一个典型的场景是虚拟机或云主机，如何优化网络IO呢？下面要介绍的是纯软件的优化方案：RPS & RFS， 这是在2.6.35内核加入的由Google工程师Tom Herbert开发的优化补丁。它的工作原理如下图所示：

![multiqueue](/res/201605-linux_net_parallel/linux_net_rps.png)

RPS是工作在NAPI层(或者说在Soft-IRQ处理中接近入口的位置)的入流量分发机制，它利用了前面提到的Per-CPU的backlog队列来将数据包分发到目标core上。默认的分发算法与多队列机制类似，也是使用IP,Port四元组的哈希来映射到某一个core。与多队列机制类似，若是流量来自少量IP:Port的场景，负载将无法很好地均衡在多核上。我们目前在AWS虚拟机上普遍配置启用了RPS，优化效果还是非常明显。

配置RPS的方法也很简单：

	echo ff > /sys/class/net/eth0/queues/rx-0/rps_cpus

更详细的配置说明可[参考这里](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Performance_Tuning_Guide/network-rps.html)

如何观察RPS的分发效果呢？由于RPS分发会多做一次Soft-IRQ调度，我们可以通过观察Soft-IRQ的统计接口来观察调度效果：

	cat /proc/softirqs | grep NET_RX

前面我们说到在没有多队列网卡的服务器，RPS可以发挥重要作用，那如果已经有多队列网卡了是否还需要RPS呢？根据我目前的经验来说，一般情况下有队列网卡的环境下配置RPS不会再有明显的提升。但我认为仍存在一些情况结合RPS是有意义的，比如队列数明显少于核数，再比如某些RFS（下面会介绍）可以优化的场景可以打开RPS+RFS。

如果你有兴趣看一下RPS的关键内核代码，可以[查看这里](http://lxr.free-electrons.com/source/net/core/dev.c#L4202)。

这里也[有篇文章](http://simohayha.iteye.com/blog/720850)介绍了一些内核实现细节可作参考。

#### RFS

RFS是在RPS分发机制基础上的一个扩展。它尝试解决这么一个问题，即然我在软件层面做数据包的分发，能不能比硬件多队列方案的近似随机的Hash分发方式更智能更高效一些呢？比如按Hash分发的一个问题就是，准备接收这个数据包的进程所在core很可能跟按Hash选择的core不是同一个，这样会导致cache miss及cache line bouncing，在多核高并发场景这对性能影响会十分可观。

RFS尝试优化这个问题，它尽力将收到数据包分发给接收它的进程所在的core上，先看一下原理图：

![rfs](/res/201605-linux_net_parallel/linux_net_rfs.png)

首先RFS会维护一张全局的路由表（图中SockFlowTable），表中记录了一个FlowHash(四元组的Hash值)到对应CPU核的路由项。表项怎么建立呢？是在进程调用某socket的recvmsg系统调用(也包括recv/recvfrom)时，将该socket的FlowHash值（由最后一次收到包的FlowHash决定）与当前的CPU核关联起来。在RPS做包转发时，实际它会先判断是否启用了RFS，并且能找到有效的RFS路由项，否则的话仍使用默认RPS逻缉进行转发。

另外RFS还维护一张Per-Queue的局部路由表（图中Per-Queue FlowTable），它有什么用呢？主要作用是为了在全局路由表发生变化时，避免原路由路径上的包还没有被完全处理完而导致的乱序。它的原理并不复杂，在局部路由表中会记录某FlowHash(实际实现是FlowHash的hash)最后一次包转发时的关联CPU核，同时会记录当时该核对应的backlog队列的队尾标号(qtail)。当下次转发该flow上下一个包时，如果全局路由给出的CPU核发生了变化，则判断当前backlog队列的队首标号是否大于qtail，如果是说明上一次转发的包已经被处理完了，可以安全地切换到全局路由给出的新的CPU核，否则的话为了保证有序仍选择上一次使用的CPU核。

从原理上可以看出，在每个socket有唯一的进程/线程处理时RFS会有较好地效果，同时，对于在同一进程/线程内多路复用操作多个socket的场景，建议结合绑定进程/线程到固定CPU核的方式可以进一步发挥RFS的作用（让转发路由规则固定，进一步减少cache line bouncing）。

RFS的配置也比较简单，有两处，一个是全局路由表的大小，另一个是局部路由表的大小（一般设为前者大小/RX-Queue数），如下例：

	echo 32768 > /proc/sys/net/core/rps_sock_flow_entries
	echo 2048 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt

更详细的配置说明可[参考这里](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Performance_Tuning_Guide/network-rfs.html)。

如果有兴趣看一下它的实现，可从[这里(记录路由)](http://lxr.free-electrons.com/source/net/ipv4/af_inet.c#L762)和[这里(查询路由)](http://lxr.free-electrons.com/source/net/core/dev.c#L3481)入手。

这里也[有篇文章](http://www.pagefault.info/?p=115)介绍了实现可供参考。


#### XPS

作者也是Google的Tom Herbert，内核2.6.38被引入。XPS解决的是一个在多队列网卡场景下才存在的问题：默认情况下当协议栈处理到需要向一个网卡设备发包时，如果是多队列网卡(有多个TX-Queue)，会使用四元组hash的方式选择一个TX-Queue进行发送。这里有一个性能损耗是，在多核的场景下，可能会存多个核同时向一个TX-Queue发送数据的情况，因为这个操作需要写相应的tx\_ring等内存，会引发cache line bouncing的问题，带来系统整体性能的下降。而XPS提供这样一种机制，可以将不同的TX-Queue固定地分配给不同的CPU集合去操作，这样对于某一个TX-Queue，仅有一个或少数几个CPU核会去写，可以避免或大大减少冲突写带来的cache line bouncing问题。

设置XPS非常简单，与RPS类似，如下示例：

	echo 11 > /sys/class/net/eth0/queues/tx-0/xps_cpus
	echo 22 > /sys/class/net/eth0/queues/tx-1/xps_cpus
	echo 44 > /sys/class/net/eth0/queues/tx-2/xps_cpus
	echo 88 > /sys/class/net/eth0/queues/tx-3/xps_cpus

可以注意的是，根据原理，对于非多队列网卡设置XPS是没有意义和效果的。如果一个CPU核没有出现在任何一个TX-Queue的xps\_cpus设置里，当该CPU核对该设备发包时，会退回使用默认hash的方式去选择TX-Queue。

如果你对它的实现有兴趣，可以从[这里看起](http://lxr.free-electrons.com/source/net/core/dev.c#L3203)。

这里是一篇[原作者对此的简介](https://lwn.net/Articles/412062/)。


#### SO_REUSEPORT

前面我们讨论的都是协议栈偏底层的并行优化，然而在上层也就是socket层，同样有一个重要的优化补丁：SO\_REUSEPORT socket选项（注意不要与SO\_REUSEADDR搞混）。它的作者还是Tom Herbert～(本文应感谢该伙计:)，在内核3.9被引入。

它解决了什么样的问题呢？考虑一个监听唯一端口的TCP服务器，如果想利用多核并发以提升总体吞吐，需要考虑使用多进程/多线程。一个简单直接的方法是多个进程竞争accept监听socket，你可能有经验这种方法的一个缺陷是，各个进程/线程无法保证负载均衡地accept到新socket。直接解决这个问题可能需要写比较麻烦的workaround（比如我们曾使用连接数表+sched\_yield的方法来保证负载均衡）。还有一个流行的处理模式是，使用一个线程负责listen和accept，然后将socket负载均衡地dispatch到一个worker线程组，每个worker线程处理一个socket子集的IO。这种模式对于长连接服务还是比较适合的，但如果是有大量connect请求的短连接场景，单个线程accept将可能成为瓶颈。解决这个问题可能又需要考虑使用复杂的多线程竞争accept的方式，但依然有socket访问竞争、cache line bouncing等效率问题。

对于UDP服务器，也有类似的问题，单进程读容易达到单核瓶颈，多进程竞争读又会有一定的性能损耗。多进程竞争读的原理如下图所示：

![reuseport](/res/201605-linux_net_parallel/linux_net_reuseport.png)

SO\_REUSEPORT很好地解决了多进程读写同一端口场景的2个问题：负载均衡和访问竞争。通过这个选项，多个用户进程/线程可以各自创建一个独立socket，但它们又共享同一端口，该端口的流量默认按四元组hash的方式分发各socket上（最新内核还支持使用bpf方式自定义分发策略），思路是不是非常熟悉。原理示意图如下：

![reuseport](/res/201605-linux_net_parallel/linux_net_reuseport2.png)

使用此方式，TCP/UDP服务器编程模式都非常简单了，多进程/线程创建socket设置SO\_REUSEPORT后bind，后面像单进程一样处理就可以了。同时性能也可获得明显提升，我们较早前一个经验是UDP改造后qps提升一倍。


如果你对它的实现有兴趣，可以从[这里(UDP)](http://lxr.free-electrons.com/source/net/ipv4/udp.c#L614)和[这里(TCP)](http://lxr.free-electrons.com/source/net/ipv4/inet_hashtables.c#L238)看看源码。

这里有一篇介绍得也比较详细的[文章](https://lwn.net/Articles/542629/)

#### 总结

本文介绍了Linux内核关于网络IO并行化的一系列技术(多队列、RPS、RFS、XPS、SO\_REUSEPORT)，它们是在协议栈不同的层面，但都使用了类似的方法提升了网络IO的并行性，并尽量减少了cache line bouncing。这些出色的工具可以帮助我们在任何Linux平台上构建高性能的网络服务器。


