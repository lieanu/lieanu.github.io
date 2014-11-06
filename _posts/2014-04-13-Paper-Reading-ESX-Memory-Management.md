---
layout: post
title: Paper Reading - Memory Resource Management in VMware ESX Server
catagories: [ReadingNotes]
tags: [ESX, Memory, Paper, OS]
---

##1 Introduction

虚拟机对服务虚拟化来说,看起来特别有吸引力.每一个虚拟机,都看起来跟其它虚拟机物理隔离.虚拟机同样可以很方便的抽象出服务器的工作量,因为它把包括用户层应用和内核模式的OS的整个系统的运行状态,都封装了起来.

在很多计算环境中,单独的服务,都没有充分利用计算资源,从而允许它们做为虚拟机整合到一个单独的物理服务器上,而其带来的性能惩罚并不大.类似这样,许多小服务可以被整合到一个稍大些的机器,从而方便管理,并减小开销.理想情况下,系统管理员应该灵活的使用内存,处理器和其它资源,受益于统计复用(statistical multiplexing)的同时,还可以根据VMS的重要性不同,来提供资源上的保障.

数十年来,虚拟机同样被用于使不同的OS,并行的跑在一个单独的硬件平台上.VMM是一个软件层,它虚拟化硬件资源,并提供一个虚拟的硬件接口.比如,VM/370虚拟机系统,支持多种并行的虚拟机,每一个虚拟机,都认为它是原生态的跑在IBM System/370硬件架构上的.最近更多的研究,比如Disco,都重在关注使用虚拟机来提供高扩展性和容错性,使商业操作系统跑在大型的内存共享的多核处理器上.

VMware ESX Server就设计用来在虚拟机之间复用硬件资源的一个轻量级的软件层.目前的系统已经可以虚拟化Intel IA-32架构.现在Microsoft Windows2000 Server和Red Hat Linux7.2已经在上跑了.ESX的设计与Workstation大不相同,Workstation使用宿主机的虚拟机架构,利用了一个预先已经存在的操作系统来做到对I/O设备的可插拨支持.比如Linux宿主的VMM截VM内读虚拟磁盘扇区的请求,然后发出一个read()的系统调用到Linux宿主系统,来取回相应的数据.与之对应,ESX直接管理系统硬件,提供性能更高的I/O,完全掌控整个系统的资源.

ESX不对操作系统做任何改动,IBM改了,甚至Disco原型,也需要对IRIX内核进行一点改动.

这篇文章介绍了ESX1.5在内存管理上使用的一些新的机制和策略.高层的资源管理策略可以根据指定的参数和系统装载,为每一个VM分配一块内存.在多个VM之间,动态共享同一个页,可以很大程度的减少系统的内存压力.

##2 Memory Virtualization

虚拟机里跑的Guest OS希望像真实硬件一样的从0开始的地址空间.ESX给每个VM营造这种错觉,通过增加额外的地址翻译层来对物理内存进行虚拟化.借用DIsco中的术语,machine address真实的内存硬件.physical address是通过软件抽象给虚拟机的真实内存的错觉.

ESX为每个VM维护一个叫做pmap的数据结构,它负责把"physical"页(PPN)翻译成机器页(MPN).那些操纵Guest OS的页表和TLB内在的VM指令被拦截,防止它更新真正的MMU状态.不同的影子页表(包含虚拟到机器页的映射),用来保障物理页到机器页的一致性.因为TLB硬件为缓存虚拟到机器地址的翻译,所以这种方法允许正常的内存访问,而不需要额外负载.

内存系统的额外迂回层,也非常强力.Server可以通过改变它的PPN到MPN的映射,实重映射一个"physical"页,这种方式对虚拟机完全透明.Server同可以监控和插入Guest的内存访问.

##3 Reclamation Mechanisms

ESX支持对内存的overcommitment,来实现更高层程度的服务器整合,而不是简单它的进行静态分区.overcommitment的意思是配置给虚拟机的内存总数,超过真实的内存大小.系统基于配置参数和系统载入情况自动的对VM的内存分配情况进行管理.

每个虚拟机都给它们一个我拥有整个物理内存的错觉.max size.商业系统都不太支持动态的改变内存大小,Guest OS启动,内存大小基本保持不变.当内存没有被过度使用的时候,虚拟机可以给它分配MAX SIZE的内存.

###3.1 Page Replacement Issues

当内存overcommitted了,ESX必须使用某些机制来对某些的VM的内存进行回收.早期虚拟机系统使用的标准的方法是把一些物理页swap到存储上.Meta-level 页替换策略:虚拟机系统必须决定哪从此个VM撤回内存,还有哪些特定的页需要回收.

相对未通知的资源管理决策?(relatively uninformed resource management decisions).在每个VM内哪个页最没价值,这只有GuestOS知道.虽然如今我们不缺乏聪明的页替换算法,但这个问题确实是个难题.一个复杂的meta-level策略,可能引起性能异常,因为与GuestOS中的内存管理策略的无意识的交互作用.

事实上,页高度对Guest OS透明,可能导致double paging的问题.假设,meta-level policy选择了一个页进行回收,并换出.如果Guest OS内存很紧张,它可能选择同样的页,把它写到它的虚拟pagin device上(不是虚拟存储Disk?).这会导致system paging device里的页内容不对了,被写到了virtual paging device上.

###3.2 Balloning

理想情况下,VM的内存被回收了一些,那么它被配置的内存应该也要相应的少一些.ESx使用Ballooning的方法.

把small balloon module做为了伪设备驱动或内核服务载入到Guest OS中去.它们跟这个Guest没有外部接口,通过私有信道进行通信.当Server需要回收内存时,它通知Driver "inflate"(通过在VM内分配pinned物理页).VM会赶到内存紧张,把一些页换到磁盘上.

膨胀气球,增加Guest OS的内存压力,导致它调用自己的内存管理算法.当内存足够时,Guest OS从它的free list,返回内存.内存不足时,它必须回收空间,来满足驱动的内存分配请求.Guest OS决定哪些特定的页被回收,如果必要,把它们换到Virtual Disk上.Balloon驱动将每一次分配的物理页号告诉ESX Server.然后再回收相应的machine page.

虽然Guest OS不接触它分配给driver的任何物理内存.ESX不依赖于这个属性.当Guest的PPN is ballooned,系统会给它的pmap入口做个注释,然后deallocate相应的MPN.任何后序的进入这个PPN的请求,都被产生一个错误.

我们的balloon驱动,每秒对Linux, FreeBSD, Windows等操作系统进行轮疚一次,以获取目标的Balloon大小,为了避免Guest OS压力过大,会对allocation rate进行一定的限制.

未来支持内存热插拨的Guest OS可能需要额外的粗粒度的ballooning.

dbench benchmark, fileserver 40clients.ESX running on Dell Precision 420. VM running Red Hat Linux 7.2.
结果表明Balloon技术的可行性.

Ballooning 技术的限制:Guest OS启动时,balloon driver是不可用的,等等.

###3.3 Demand Paging

ESX首选ballooning技术回收内存,把它做为常见情况的优化措施.当ballooning不适用时,系统回退到页转换机制.内存在没有Guest的参与下,转换到磁盘上.

ESX的swap程序,从更高层的策略模块接收每个虚拟机的swap levels information.它管理候选页,共同的通过异步的方式把页换到Disk的一个交换区上.

必须有那么一个页置换策略来防止3.1中提到的Guest OS的内存管理算法的Double paging的问题.

##4 Sharing Memory

如果不同个VM上跑着相同的Guest OS,应用也相同 ,组件也相同,或包含的数据相同.ESX利用共享的办法,来减少内存的消耗.

###4.1 Transparent Page Sharing

Disco引入transparent page sharing,做为减轻页拷贝冗余的方法(code, read-only data).当识别出拷贝行为时,多个guest的物理页都映射到同一个机器上.如果有写的动作,再通过缺页异常的方法,做一个拷贝.

但,Disco需要对OS代码进行改动,来识别冗余拷贝.比如,bcopy()下个钩子,来实现虚拟机之间的文件缓存共享.有些共享,还需要非标准的或受限的接口.o

###4.2 Content-Based Page Sharing

在ESX这个环境里,对Guest OS进行修改,这不现实.改动API也不可接受,ESX使用一个完全不百的页共享方法.基本的想法是根据拷贝页的内容进行识别.两个优势:1.不用修改Guest OS的代码.2.more opportunities for sharing.

把每一页的内容与其它页的内容对比的开销很大,O(n2).使用Hash表.

把Page的内容做Hash,做为查找的关键字放到Hash表内.Hash表的每一项都做了COW的记号.如果新的页的Hash值在Hash表里匹配到了值,那么它们应该是一样的.

如果找到相同页,然后使copy-on-write机制来共享页,这样冗余的copy动作都可以省掉了.如果有写的动作,触发缺页异常,再做一个新的拷贝即可.

如果没有匹配到,一个选择是在页上标记COW.然后这个方法过分简单了,带了非预期的副作用,它使得扫描过的每一个页,都做了COW标记.导致不必要的负载.一个优化手段是,非共享的页,不做COW标记,但是tag一个Hint,如果将来另一个页匹配到了,这个hint page再做Hash.如果Hash值不同,那么Hint page已经改变了,把老的Hint移除.如果Hash值依旧一样,那么搞一个全面的比较,如果仍相同,那么就让页共享.

更高层的页共享策略,控制什么时候什么位置来扫描副本.一个简单的选项是以一个固定的速度扫描增加的Pages.otherwise-wasted idle cycles.

###4.3 Implementation

一个全局的Hash表,表中每一个frame有16字节.同享的frame(shared frame)由hash value, MPN, count, chaining组成,hint frame类似,但是Hash value是截断的,主要是让点地方给对应Guest Page的引用,由一个VM标识符和PPN组成的.其所占空间,不会超过0.5%的系统内存.

与Disco的页共享实现方法不同,它为每个共享页,维护了一个backmap,ESX使用一个简单的引用计数.每个Frame仅存储16bit的Count,一个单独的溢出表来存储更大数的Frames.这允许高度共享的页可以更紧密的表示.

拿一个fast, high-quality的Hash函数来给每个扫描过的页,做64-bit的Hash.想发生碰撞几乎不可能.因此,系统可以假设所有的共享页有不同的Hash值.

目前ESX的页共享实现随机扫描Guest的页.虽然使用更复杂的方式也未尝不可,但这个方法足够简单和高效.配置选项控制每个VM的最大数和扫描速度,确保不会使用太多的CPU负载.

为了对这个实现进行评估,我们通过对reclaiming memory和系统性能的负载进行了实验."best case",包括很多同一类型的VM,来证明,当可能的共享存在时,ESX可以实现对很大一部分的memory的reclaim.然后额外收集了,部署给真实用户使用的数据.

相同配置的虚拟机,每个上面跑RedHat Linux 7.2,40MB物理内存.每次试验跑1至10个VMS,拿SPEC95 benchmark测30分钟.

单独一个虚拟机也回收了不少共享内存,5MB.55%的共享内存,都是zero page.没有优化,共享的内存是线性增加的,符合期望.有优化的情况下,随着VM数的增加,共享的level达到67%,每个VM三分之二的内存.

更多不同的工作负荷,其共享度也更低.尽管如此,很多真实世界的Server大都使用相同的Guest OS来跑类似的程序.

##5 Shares vs. Working Sets

根据服务重要性,选择牺牲不太重要的VM,节省内存资源,来使目标VM达到性能最大化.

###5.1 Share-Based Allocation

在proportional-share框架中,资源的权限被share封装,它们属于那些消费资源的client.client根据它的共享分配,按比较的消费资源.share表示相关的资源权限,它取决于一个的资源的总的共享数.client allocation在资源紧张的情况下,可优化的降级,资源剩余时,可按比例受益于额外的资源.

随机化和确定性算法都提出用在proportiona-share allocation of space-shared resource. Dynamic min-funding revocation 算法简单高效.当一个client需要更多空间时,置换算法选择一个受害者client,使其撤回它已经分配的一些空间.内存从那些共享内存最小的client上收回.经济学的比喻,把memory资源从那些出价低的client上,撤到出价高的client上.

###5.2 Reclaiming Idle Memory

pure proportional-shared 算法的最大的限制是它没有incorporate任何已使用内存或工作集的信息.根据指定的比率,内存被分隔.但是一些很闲的client非常浪费的囤积内存.性能隔离和高效利用内存常常是互相冲突的.

ESX提出idle memory tax来解决这个问题.Basic idea:idle page要比在用的page有charge more.内存不够使时,就从这些没有太多使用它们的内存的那些client上回收.税率指定了空闲页的指数最大的client.

Min-funding revocation 使用一个调整后的shares-per-page ratio来进行扩展.a client with S shares 分配了P pages, 活动的有f, 调整后的每页共享率为p

```
p = S/(P * (f + k*(1 -f)))
```

ESX 的空闲内存税率是可配置的,默认是75%.这允许大部分的idle memory可以被收回,以提供了一缓冲来应对快速的working set的增长,从而掩盖系统回收动作的延迟(ballooning, swapping).

###5.3 Measuring Idle Memory

内存空不空闲不好分别,一个选择是使用Guest OS的原生态的接口来提取信息.但这种方法不切实际 ,因为不同的Guest使用不同的度量指标.这些指标更趋向于每个进程的工作集.

ESX使用统计抽象的方法,直接获取VM工作集的估值,而不需要任何Guest的参与.每个VM是一个独立的样本,使用在VM执行时间单元里定义的可配置的样本周期.在每个样本周期开始时,随机选取n个物理页.每个样本页通过使缓冲映射与PPN之间的关系无效,来记录,就像TLB entries和虚拟化的MMU状态.f = t/n.

tradeoff 额外开销和精确度.默认情况下,ESX每30秒抽样100个页.也就是100的缺页异常,这对于整个系统来说,一点都不大,为保证精确性,每30秒触发100个缺页异常是完全值得的.

slow moving average, fast moving average. Maxiumum.

##6 Allocation Policies

ESX基于share-based entitlement和上一章所描述的算法,为每一个VM计算分配的内存.这些都通过ballooning和页置换机制实现.页共享机制运行在后台,可动态的减少系统内存使用压力.

###6.1 Parameters

系统管理员使用三个基本的参数,来控制每个VM的内存分配:最小值,最大值,内存shares.最小值是保证分配给虚拟机内存的一个底线,甚至内存被过度使用的情况下.最大值是为每个VM中的GuestOS分配的物理内存.除非内存被过度使用了,VM通常分配最大值.

内存共享(memory shares), 基于proportional-share allocation策略,给VM一部分的物理内存.

###6.2 Admission Control

机器内存(machine memory)必须保留一些用于最小值保障,而且还有额外的内存开销,用于虚拟化,总数Min + overhead.overhead内存包括不同的虚拟化数据结构(pmap,影子页表).通常,VM为overhead预留32MB.4至8MB用于Frame Buffer,其它用于实现指定的数据结构.

磁盘的交换空间必须预留,用做剩余VM内存.从而确保系统可以在任何环境下,保持虚拟机内存.实际上,只有很小一部分的磁盘会被使用.类似的,内存预留被用做admission control.

###6.3 Dynamic Reallocation

为应对多种事件,ESX动态的重计算内存分配.这些事件包括:系统管理员改更改到系统级,或每个VM的分配参数.

大部分OS尝试维护一个最小化的空闲内存.比如,BSD通常在空闲内存达到5%以下时,回收内存,直到空闲内存达到7%为止.ESX使用一个类似的方法,但是使用4个值,high,soft,hard,low,每一个默认是6%,4%,2%,1%.

在high的状态时,空闲内存足够,没有回收动作.在soft状态,系统使用ballooning回收内存,当ballooning不可用时,再借助于paging.在hard状态,系统依赖paing来强制回收内存.还一种很少见的情况是,内存瞬间低于low的值,系统会持续使用paging来回收内存.

在所有的内存回收状态,系统为VM计算target allocations来驱动空闲空间数在high值之上.回收内存后,系统返回上一状态.这种滞后状态,防止了快速的状态变化.

##7 I/O Page Remapping

现代的IA-32处理器支持物理内存扩展(PAE)模式,允许硬件使用36bit的地址,从而达到64GB的内存范围.然而,许多使用DMA做为I/O传送的设备,只能使用一部分.比如一些32-bit网卡只可以进行4GB的内存寻址.

问题是上面这个问题,但文章有些老了,现在的64bit支持,应该不需要这一章用到的技术了吧.

##8 Related Work

* Disco, Celluar Disco.多处理器,共享内存,跑多个IRIX的实例.
* Distinction with Workstation. Workstation使用宿主的架构,实现不同桌面系统的可移植性.
* 跑商业操作系统不需要改动内核.
* Ballooning技术.self-paging技术.
* Content-based page sharing.
* 内存压缩技术(IBM MXT).
* page-remmapping.
* 抽象统计方法.
* proportional-share allocation.

---

[You can download this paper **HERE**][1]

[1]: {{site.baseurl}}/papers/os/esx_waldspurger.pdf
