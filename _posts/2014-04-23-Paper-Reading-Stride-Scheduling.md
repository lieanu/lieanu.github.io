---
layout: post
title: Paper Reading - Stride Scheduling Deterministic Proportional-Share Resource Management
catagories: [ReadingNotes]
tags: [stride scheduling,  Paper, OS]
---

> 没有细读...

stride scheduling来源于lottery scheduling,与Lottery相比,它相对吞吐率更精确,反应时间更短.

##问题与现状

缺乏通用的方法,来支持灵活的,responsive control over service rate的调度需求. 精确的控制relative computation rates很重要,除了在CPU调试方面,还可以把这种方法应用到数据库,网络,音视频应用等上面.

以前提出过一种Lottery scheduling的方法,这是按比例共享资源管理的.stride scheduling也是同样的一种支持灵活的资源管理的方法.作者的贡献之一,它是中跨平台的,可以将它用在网络上.之二,还提出了一些新的控制资源的方法,比如改变资源的分配比例,在client之间传送资源的权限.之三提出了一个新的分层次的stride调度算法.

##Stride  Scheduling

它是用于时间共享资源的确定性的分配管理机制.一个标准的时间片叫quantum,资源的权限用ticket的数量表示.活动状态的client的吞吐率就以ticket的相对数量多少来表示.(n * t/T)来表示,其中n是总资源数,t是当前拥有的tickets,T是总的Tickets.

stride scheduling提供更强的确定性保障.stride scheduling的relative error不会大于1.lottery scheduling是O(n(1/2)).stride比lottery更准确,随着分配数增加,而错误情况确不增加.而分层的调度更少O(lg n).

###基本算法

时间间隔用stride表示,stride越小的client就调用的越频繁.比如某个client只有另一个的一半的stride,那它会比那个client快2倍.还有个概念叫做pass,表示它走了多长.tickets表示client相对于其它client分配到的资源.stride与tickets成反比.

算法描述:
首先选择最小pass的client执行,pass优先于stride,如果pass一样的有多个,选择stride最小的,如果两个都一样的有多个,随便选一个,但最好还是定个顺序.

在allocate之前,应该init.即

```c
stride = stride1/tickets //stride1表示一个大整数,要合理确定,文章里给的是1<<20,即1M.
pass = stride
```

为精确的表示,浮点数也可以使用.(但不建议使用,浮点运算还是要比整数运算慢很多的,操作系统要注重性能),文章使用的是一个大的整型常量,即stride1,stride如果为stride1,即表示,它手上的票只有1个.

分配算法的性能,主要依赖于数据结构.优先队列(priority queue)就不错,O(lgn).跳表也能达到O(lgn).

###Client动态参与进来

刚才的算法还不支持这个功能,稍做拓展:global_tickets表示总的ticket数,global_pass表示当前的scheduler的pass.client加入之前,先算global_pass,再算该client的pass,然后将之加到优先队列里.

```c
global_pass_update();
c->pass = global_pass + c->remain;

global_tickets_update(c->tickets);
queue_insert(q,c);
```

新的client加入或离开,时间复杂度为O(lgn).

###动态的改动ticket

当client的分配,动态的从ticket改到ticket1时,它的stride和Pass同样需要重新计算.stride1好算.pass1要先看剩下的,用remain表示,再根据stride1调整.

```c
//先把cient拿掉
client_leave();

//算新的stride
stride = stride1/tickets;

//算新的remain
remain = (c->remain * stride) /c->stride;

//更新状态
c->strides = stride;
c->tickets = tickets;
c->remain = remain;

//再把client加进去
client_join();
```

其时间复杂度,同样为O(lgn).

##灵活的资源管理

主要有ticket转移,ticket膨胀,ticket流通三种方法,想法均来源于经济学的原理.

###Ticket转移

ticket transfer是把ticket从一个client传送到另一个client.比如在RPC调用中,client就可以把自己的ticket传给Server,以便于Server关于这个Client的线程获得更多的执行机会.如果一个服务器上有多个Server,就可以根据Client的连接数及需求,动态的给相应的Server分配更多的资源.

要注意的是,不能把A所有的tickets都传给B了,因为这样的话,A的stride是无穷的,因为它除0了,所以要给A按比例留下那么一点Tickets.

###Ticket膨胀

Tickets Inflation本质上就是dynamic ticket modification.client根据自己的需要来选择膨胀还是缩小它的tickets,来获取更多或者释放它的资源.

tickets Inflation的方法,可以用在可信的client上.因为它把资源的权限都交给client了,所以还是很危险的,为了尽量避免这种危险,作者借用经济学的方法,提出了货币抽象的概念.

###Ticket Currencies

用这个方法,来解决Ticket Inflation的信任问题.用ticket表示货币.通过通货膨胀的方法,来改变client对资源的拥有率.

##相关的工作

###Rated-based Network Flow Control

stride调度算法与之类似(Zhang 's VirtualClock algorithm).通过Virtual Clock来实现控制,对应pass.Virutal tick就对于stride.

Virtual Clock算法与weighted faire queueing(WFQ)算法近似.相关的算法还有packet-by-packet generalized processor sharing(PGPS)算法.

###Proportional-Shared Schedulers

如今已经有一些确定性的方法为proportional-share processor scheduling提出,但对动态改动或分布环境支持不太好,均不能提供灵活的资源管理支持.

###Priority Scheduler

传统的操作系统一般使用优先级模式来实现进程调度.但它不提供proportional-share control over relative computaton rates支持.decay-usage 调度很难理解.Faire share调度方法可以使用户在很长的一段时内,获取公平的机器共享.这些算法通常都很复杂,需要周期性的监控,还有复杂的动态优先级调整策略.

###实时调度系统

实时调度被设计用在时间关键的系统中,包括一些航天和军用的应用.rate-monotonic scheduling, earliest deadline scheduling.实时调度的应用环境还是比较苛刻的,stride调度主要是能用在一个通用的环境中.

###微观经济调度

微观经济调度是基于资源分配在真实经常系统中的比喻.Money封装成资源的权限,价格机制用来分配资源.escalator algorithm做单处理器的调度,Spawn 系统依赖于竞标者一直是线性的增加它的bids.

Stride调度方法,也兼容基于市场的资源管理哲学.其灵活的资源管理的想法同样借用于经济学.

---

[You can download this paper **HERE**][1]

[1]: {{site.baseurl}}/papers/os/waldspurger.weihl.pdf
