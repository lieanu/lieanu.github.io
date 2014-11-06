---
layout: post
title: Paper Reading - mTCP - A Highly Scalable User-level TCP Stack for Multicore Systems
catagories: [ReadingNotes]
tags: [mTCP, OS, tcp/ip]
---


1. Introduction
----------------

短TCP连接现在很普遍,长TCP连接(视频服务等)消耗的是带宽,短连接消耗的是TCP连接数.在一个大型蜂窝网内,90%以上的TCP流量小于32KB,一半以上小于4KB.

扩展短连接处理效率,不仅对流行的面向用户的在线服务有重要意义,同时对一些后台系统(如memcached clusters)和middleboxes(ssl 代理等)同样很重要,因为这些服务都需要尽高速的来处理短TCP.尽管近来这方面发展迅速,但高速的TCP事务处理,依然是一个挑战.


Linux的TCP事务处理的峰值是每秒0.3million个,而如今I/O却达到10million个每秒,很明显,Linux系统内核的处理,已经是一个瓶颈.

在这之前的研究,都关注在syscall的高负载或多核系统的实现上导致的资源冲突.之前的方法,彻底的改变了I/O的抽象,来缓冲syscall的消耗.这种方法的实际限制,是它需要对内核做很多改动,并迫使应用端重写.

本文,提出一种不需改动现有代码方法.[mTCP][1]: user-level TCP stack.目标:

1. TCP stack的多核扩展;
2. 易于使用,应用便于移植到mTCP;
3. 易于部署,不需要内核代码的改动.

关键方法:batch of packet-level and socket-level events.

1. packet-event & socket-event batching 的整合.性能提升:33%SSLShader, 320%lighttpd
2. Done at the user level. 不需更改内核. BSD-like socket.(accept() --> mtcp_accept())

2. Background and Motivation
-----------------------------

现存问题:

###2.1 Limitations of the Kernel's TCP Stack###

four major inefficiencies in Linux TCP Stack:

1. lack of connection locality
2. shared file descriptor space
3. inefficient packet processing
4. heavy system call overhead

**Lack of connection locality**

为在多核机器上提升性能,许多应用都是多线程的,但是它们共享一个socket,所以要通过锁的机制来获取socket,因此对性能影响严重.内核处理TCP连接可能与应用端代码处理的不同(lack of connection locality)导致额外的负载(CPU 缓存不命中,cache-line sharing)

```
accept queue --> per core
SO_REUSEPORT (linux kernel 3.9.4)
```

**Shared file descriptor space**

在支持posix的操作系统中,文件描述符fd是进程内共享的,分配新的socket时,就需要查找最小可用的fd号.在一个任务繁重的server中,多线程之间需要额外的锁的开销.在socket上使用fd,需要额外的检查vfs的负载.MegaPipe通过分隔socket fd和一般文件的fd,来减少这种开销.

**Inefficient per-packet processing**

per-packet内存分配和DMA的开销.NUMA-unaware memory access和heavy data structures(sk_buff)是处理小包的主要瓶颈.采取批处理的方法,来减少开销.

```
user-level packet I/O libs
```

**System call overhead**

syscall user/kernel mode switching.频繁的syscall调用导致处理器状态污染(顶层cache,分支预测表),带来性能上的损失.Solution: syscall batching, efficient scheduling.但以上两种方法,均需要对syscall接口和语义的改动.

###2.2 Why User-level TCP###

很少previous designs真正解决了上文所提到的内核态中的效率问题.例如MegaPipe,可以称为性能最好的系统之一,需要花费大部分的cpu周期在内核态(80%).所以CPU周期尚未被有效的利用起来.Linux是mTCP的4倍.

8-core, lighttpd, 大量并发的TCP连接(8K-48K),32G memory, 10G NIC. client(ab v2.3)

MegaPipe做了几乎所有的优化策略,除了重用内核态中的packet I/O和TCP/IP processing.

文中的结果表明,Linux和MegaPipe的CPU周期的内核态占用率达80%-83%.锁,缓存管理,频繁的用户态和内核态切换是主要原因.因此kernel以及TCP协议栈实现,是主要瓶颈.而mTCP是Linux的4.3倍以上.因其内核态中和TCP栈上消耗的CPU周期更少.

***The motivation:***

* 是否可以设计出一种用户态的TCP协议栈,并整合现有的优化策略于一体?
* 如果搞定这样一个系统,性能方面的提升有多少?
* 现有的packet I/O libs能否用到TCP stack上,用于性能提升.

***为何用户层的TCP协议栈更attractive***

* 把TCP stack从复杂的内核中解放出来.
* 把已有的high-performance packet I/O library直接拿来就用.
* batch processing, 无需对内核代码进行更改.无需系统状态切换.
* mTCP stack向后兼容,支持BSD-like socket接口.

3.Design
----------

目标是在多核系统上高扩展性,并保持向多线程,事件驱动的应用兼容.

###3.1 User-level Packet I/O Library###

Polling(轮询)极度浪费CPU周期.我们需要多网卡之间高效的多路复用.比如发数数据包时,不希望TX队列被阻塞住,减少重传.

mTCP扩展了PacketShader I/O engine(PSIO)来支持event-driven packet I/O interface.利用RSS(把收到的包根据flow放到不同的RX队列中,flow层的亲缘关系来最小化cpu内核的资源竞争).允许mTCP线程同一时间从多个NIC接口的RX/TX队列中等待事件.

PSIO减小了syscall和上下文切换的开销,还减少了包内存分配和DMA的开销.PSIO中,包是批量发送的,减少了类似于DMA地址映射和IOMMU查找的开销.

###3.2 User-level TCP Stack###

使用用户层的TCP协议栈,自然避免了syscall的开销(比如socket I/O).实现用户层tcp协议栈的一个方法:做成一个library,并做为应用层程的主线程.这种方法的局限性是TCP进程间处理的正确性依赖于从application及时唤醒TCP functions.

在mTCP中,采取创建一个独立的TCP线程来避免这种事情.Application通过Shared TCP buffers来使用mTCP library.但这种方法会产生来自于管理并发数据结构和application和mTCP线程切换的额外开销.不幸的是这种额外开销,甚至远远大于syscall(19倍).

**3.2.1 Basic TCP Processing**

mTCP线程从RX队列读取一批packets,然后把它传给TCP packet processing.每个包,mTCP先搜索对应flow hash table内的flow的TCP control block.例如:如果server端收到ACK:

1. 新连接的tcb(TCP control block)入队accept queue.
2. 为监听的socket,产生一个read事件.
3. 如果新数包到达,拷贝payload到socket的read buff,并且入队一个读事件到event queue.

处理完一批包后,刷新application的event queue.

4. 唤醒application.
5. 向多个flows写回复,无需上下文切换.
6. 入队tcb到wirte queue
7. 集中所有有数据发送的tcbs, 把它们放到send list内
8. 将这一批包,传给TX queue.(by a packet I/O system call)

**3.2.2 Lock-free, Per-core Data Structures**

每个core,都给分配了所有的资源.在application和mTCP之间使用lock-free 数据结构.

Thread mapping and flowlevel core affinity: flow-level core affinity in two stages: 首先,packet I/O层使用RSS,确保TCP连接在多个核之间的负载均衡.其次,mTCP为每个application线程产生一个TCP线程,并将之放到同一个物理CPU核中.使用同一个CPU缓存,无需cache-line sharing.

Multi-core and cache-friendly data structures: 为每个TCP线程保持大部分的数据结构:flow hash table, socket id 管理, tcb池, socket buffers.当有数据结构需要共享(application , mTCP), 保持所有数据结构在每个核上的局部性,使用lock-free数据结构(生产者消费者).我们来维护write,connect, close, accept queue.

经常访问的数据结构,尽量小,这样能最大化的利用CPU的缓存.cache-line大小对齐.如把tcb分成两部分,第一层的结构64bytes,放那些最常访问的域.

为减少频繁的内存分配动作,为tcbs和socket buffers在每个核上独自分配一个内存池.通过huge pages来减少TLB不命中的情况.

Efficient TCP timer management: 因为超时重传,TIME_WAIT状态, keep-alive checks等问题,TCP需要一个timer的操作.2种方法:排序表或hash表.粗粒度的timers,将tcbs通过对timeout的值进行排序,放到一个列表内.每秒,检查list,并处理到期的tcb.细粗度的重传timers,使用remaining time(milliseconds)做为hash表的索引.

**3.2.3 Batched Event Handling**

批量收包,产生一个flow-level event的batch.然后将事件传给application.TX端类似,mTCP通过write queue批量处理write事件.

每10Gbps的端口,使用8个RX/TX队列.每队列平均2170个mTCP线程.

并发短连接的处理,两种优化策略:

Prority-based packet queueing: 短连接中,控制包对性能影响至关重要(SYN,FIN).控制包通常很小,在大量的包中,争夺端口资源的几率小,因此延迟大.把这些类型的包放到一个单独的list中.

Lightweight connection setup: connection setup 的开销,主要在TCB和socket buffer的内存空间分配.预先分配好一个很大的内存池.

###3.3 Application Programming Interface###

该系统的主要目标之一,就是减小现有应用的移植工作.保持接口及接口语义的一致性.

**User-level socket API**:类似BSD socket的接口.accept() --> mtcp_accept(). other funcs: e.g.. fcntl() ioctl(). mtcppipe().

socket描述符空间是mTCP线程局部的.每个mTCP socket都关联一个线程上下文.这样节省了进程上下文之间的锁的开销.另外,存找最小fd也是开销,直接返回一个可用的fd即可.省节这一部分的开销.

**User-level event system**: 开发了epoll()-like的事件系统.该事件处理系统只是将多个flow的事件指量处理,不改变事件处理逻辑的语义.mtcp_epoll_wait() --> epoll_wait().

**Application**: mTCP在不改动内核代码,保持接口一致性的前提下,整合了现今几乎所有已知的优化技术.因此应用程序可以很方便的扩展使用,而无需改变程序逻辑.

因为shared TCP buffers的存在,application中的漏洞,会影响到TCP Stack.mTCP会绕过现有的Linux Kernel的一些服务,如防火墙等.如今的原型目前只支持单个应用程序.

4 Implementation
----------------

Pass.

5 Evaluation
-----------------

Pass.

6 Related Work
-----------------

**System call and I/O batching**: 频繁的系统调用是busy servers的性能瓶颈.FlexSC表明,CPU缓存污染比syscall的用户到内核态的反复切换更浪费CPU周期.通过用户和内核空间共享syscall pages来指执行system calls.MegaPipe就采取类似的方式,但它使用的是一个标准的system call接口来与内核进行通信的.

批量处理同样用于packet I/O.PacketShader I/O引擎,就是通过批量读写的方式来提升packet I/O的性能的,尤其是对小包更有明显.它主要通过减少中断,DMA,IOMMU的查找和动态内存分配的开销,来提升性能的.

mTCP通过将TCP协议栈的实现放到用户态来提升性能.它同样在packet I/O和TCP处理上支持batching.但与FlexSC和MegaPipe不同的是,mTCP不需要额外更改内核和用户层的代码.

**Connection locality on multicore system**: 将相同的连接放到同一个核上来处理,减少了核间的数据迁移和不必要的缓存污染.mTCp在flow-level和packet-level的处理上都采取这种方法.

**User-level TCP Stacks**: 先前有过将TCP协议栈从内核态向用户态移植的尝试,但是他们主要集中在提供灵活的用户层协议开发环境或安全地暴露一些内核变量给用户层.本文主要focus在多核系统上的高扩展性.

**Light-weight networking stacks**: 有些应用鉴于性能方面的考虑不使用TCP.例如memcached等Key-value对系统.它们使用RDMA或基于UDP的协议来避免TCP的额外开销.这些只是面向Datacenter的,大多数面向用户的应用,仍依赖于TCP协议.

**Multikernel**: 很多研究致力于加强OS在多核系统上的扩展性.Barrelfish和Fos就是通过per-core的方式,为每个核独立分配内核资源.核间通信采取异步消息传递的方式,而mTCP使用lock-free data structures.

**Microkernels**: Mircrokernels(微核)与mTCP类似,系统服务跑在用户层.Exokernel(外核)提供一个微内核和一个底层的可访问硬件的接口.它通过将底层硬件直接暴露给用户来提升性能.

---

[You can Download this paper **Here**.][1]
[1]: {{site.baseurl}}/papers/os/mTCP-nsdi14.pdf
