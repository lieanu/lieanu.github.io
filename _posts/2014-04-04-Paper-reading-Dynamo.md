---
layout: post
title: Paper Reading - Amazon Dynamo
catagories: [ReadingNotes]
tags: [Dynamo, Amazon, Paper, Distribute system]
---

1.Introduction
--------------

Amazon作为全球最大的电商平台之一,高峰时能达到ten millions的客流量, 因此对性能,效率,可靠性等的要求非常高,而reliability可以说是重中之重.
Amazon通过高度分散的,松耦合,而向服务的结构,提供数以百计的服务,在这种环境下,对存储的需求非常重要.服务必须在磁盘坏了,网络挂了以后,能不丢失数据的,继续为用户提供服务.
其实,Amazon这么大的平台,总有那么几台机器或网络会出现问题,这已经是正常现象,而Amazon的系统必须要把这种现象当作日常案例,在不影响性能和可用性的情况下来处理.
为meets这种可靠性和scale的需求,Amazon开发了一系列的存储技术,如S3.本文给出的是[Dynamo][1]的实现,另一个为Amazon设计的高度可用的,分布式的数据存储.

[Dynamo][1]即是这样的一个系统,它首要的是提供高度的可靠性,并在availability, consistency, cost-effectiveness 和performance之间取得折衷.
根据不同的存储需求,Amazon提供不同种类的application.application应该是flexible的,留给application designers更多选择.

其实Amazon上的很服务器,只需要primary-key的方式,而如果使用传统的关系数据库,会导致效率,规模,可用性上的问题,[Dynamo][1]根据这种需求,提供了primay-key only的接口.

Amazon采取了一系列的技术来做到scalability & availability:

* Data is partitioned and replicated using consistent hashing, consistent by  object versioning. 
* 副本之间的一致性采取quorum-like的技术和decentralized replica synchronization protocal.
* [Dynamo][1]采取gossip based distributed failure detection and membership protocal.
* 存储节点可以动态的从[Dynamo][1]上添加和删除,而不需要人工的partitioning and redistribution.

去年,[Dynamo][1]已经为Amazon的一些核心服务开始工作了,事实证明,它可以scale 节日购物季的高峰负载需求.1天有3 millions的checkouts,就能说明问题.

**本文贡献:**

TODO::

* 稍后再写 

2.Background
------------

Amazon电商平台上由数以百计的服务组成,包括从推荐系统到欺诈检测,每个服务都通过一个接口暴露出来,可从网络连接.这些服务遍及成百上千的服务器,其中有些服务是stateless,有些是stateful的.

传统的产品系统大多数把他们的状态存储在关系数据库内,但关系数据库的确不是一个理想的选择.大多数服务只需要通过primary key进行数据的存取,而不需要复杂的查询和RDBMS的管理功能.因为这些额外的功能,需要昂贵的硬件支持和技能熟练度高的人来操作,这是一个很没效率的解决办法.

[Dynamo][1]使用简单的key/value接口,高可用性(清晰定义的一致性窗口), 高效的资源使用, 简单的扩展模式来满足数据集大小和请求速度的要求.每个使用Dynamo的服务,都有自己的Dynamo实例.

###2.1 System Assumptions And Requirements###

需求分析:

* 查询模型.通过key来进行简单的读写操作,state区分不同的Keys以二进制对象存储. 没有操作跨及多个数据条目,因此不需要关系模型.这个模型是通过观察得知.[Dynamo][1]主要面向那些只需要较小存储的应用,一般来说不超过1MB.
* ACID 属性.ACID即是*Atomicity, Consistency, Isolation, Durability*.是一组用于确保数据库事务处理的可靠性的属性. 经验上看,提供ACID属性的数据存储,往往都是poor availability的, 这种现象同样也被工业界和学术界所认同.[Dynamo][1]面向那一些操作一致性更弱的应用,Dynamo不提供任何独立性保障,只允许单个的Key updates.
* Efficiency.在performance cost-efficiency availability 和 Durability guarantees之间进行权衡.
* 其它假定.[Dynamo][1]运行在Amazon内部,不具有敌对性,不用考虑安全性需求.而且,因为每个服务有自己的Dynamo实例,所以Dynamo的初始设计应该面向数以百计的存储主机.

###2.2 Service Level Agreements(SLA)###

考虑efficiency的问题,client & server是一种SLA的形式.举例来说,一个简单的SLA就是:一个服务保证自己可以在每秒500请求的情况下,在300ms内回应99.9%的请求.

在Amazon去中心化面向服务的架构内,SLAs扮演很重要的角色.因为各种服务复杂的依赖关系,一个应用的call graph有很多层.为确保效率,page rendering engine应该维持一个清晰的bound, call chain上的service必须遵守这个bound(or contract).

通用的描画SLA的方法,通常用平均值,中值和期望值来描述.采取99.9% not mean or median.(99.9th percentile of distributions)

###2.3 Design Considerations###

>**CAP 法则 **
>
>Consistency Availability Partitioned-tolerated.只能同时满足2个属性,[Dynamo][1]满足A/P两个属性.

数据副本算法很重要,是strongly一致性的保障,众所周知,网络问题与consistency和avaliability三个属性,是不可能同时具有的.(***CAP法则***)

主要从Network Partitioned和Availability这两方面来考虑的话,就有很多优化的技术.这种方法的面临的主要挑战是冲突检测与解决.

    Two Problems:
        when resolves them
        who resolves them

***When Resolves Them***

传统的一个解决办法是在write的过程中解决,保持read的简单性.因此,在这样的系统中,如果所有数据副本不能同时reach, 那么write会被reject.而Dynamo的设计目标就是提供**"Always Writeable"**的数据存储.就是说任何时刻,数据都能写进去,即使网络被Partitioned(ps.确保用户任何时间都能买得了东西啊,只管赚钱...).这种需求用Read的过程中,进行冲突解决比较好,从而来解决Always writeable的问题.

***Who Resolves them***

Data store 和application都可以干这个事.如果是Data store来做这件事,它的选择可能有限,因为它只能用例如last write wins这种类似方法来解决.另一方面,application可以感知到数据的schema,它来做这件事情更适合,比如维护消费的购物车的应用,可以选择merge冲突版本,返回一个统一的购物车.尽管这可能会复杂一点,但应该很少有application的developers会把这事推给data store来做.

**其它的key principles**

* Incremental Scalability: 可以动态的增加Node, 并对System基本不造成影响.
* Symmetry: 对称性, 即没有什么节点是特殊的,大家都一个样,没有master client primary backup之分,更便于维护.
* Decentralization: 去中心化, 对称性带来的影响, 没有控制中心,更加简单,更好的扩展性,更佳的可用性.
* Heterogeneity: 多相性,机器有老有新,大家性能不可能一样,性能多大的机器就干多大的活,这样我买台新机器加进去,就不需要再升级其它结点.

3.Related Work
--------------

这段不看了,进入一下段...

###3.3 Discussion###

1. Dynamo的"always writeable"特性.
2. 安全性的问题,所有节点都认为是可信的,即不考虑安全性.
3. 不需要支持分层的命名空间或者复杂的关系模式.
4. Dynamo是低延迟的,要求99.9%的读写操作必须在数百毫秒内搞定.

尤其是第4 点需求,要求尽量避免节点之间的路由请求,zero-hop DHT, 每个节点在本地维护路够的路由信息.

4. System Architecture
-----------------------

先瞧瞧搞定这么一个复杂的系统需要搞定哪些问题:

    Load balancing
    Membership
    Failure Detection
    Failure Recovery
    Replica Synchronization
    Overload Handling
    State Transfer
    Concurrency
    Job Scheduling
    Request Marshalling
    Request Routing
    System Monitoring
    Alarming
    Configuration Management

没搞过大项目的屌丝果断的被吓尿了,还是跟着Paper看看核心关键技术怎么解决的吧.

###4.1 System Interface###

get(key) put(key, context, object) key不解释, context将object的元数据进行编码. Key是通过MD5来hash出的128位的id(?md5不是32位么)

###4.2 Partitioning Algorithm###

采取**"consistent hashing"**的方法,将hash函数的空间看成一个环,如md5看成0 - 2^32-1一个环.N个节点随机落在这个环内,hash后的key(0 - 2^32-1),如果对应存在节点,则将数据存在节点上,如果不存在,顺着环往下找,直至找到节点.

这种方法的好处:

1. 增加一个节点,不用动所有的数据,只需要把一部分的数据做一个迁移.
2. 减少一个节点,同样.

挑战:

1. 节点的随机落点,不一定比较均匀的落在hash空间内.
2. 缺乏heterogeneity特性.意思是把每个机器的存储能力考虑成一样的了.

**解决办法**: 虚拟节点,比如节点机器不怎么样,看成50个虚拟节点.刚买的新机器,各项配置比较高,看成500个节点,然后把这些虚拟节点号随机落入hash空间内.

###4.3 Replication###

假设:5个host, 1----->2----->3----->4------>5------->1(ring, 想象成一个环)

key(k)应该存在host2上,然后在host2的接下来两个节点做副本,即将key同样在host3,host4上进行存储.

###4.4 Data Versioning###

Dynamo采取**eventual consistency的方法,即最终所有副本是一致的,不关心时效性.put()不用在所有的副本都update以后再返回.当然,这可能导致部分节点的副本还没update,我就get()了,所以get()的数据不一定是最新的.如何解决?这个问题4.5节再关注.

Dynamo采用vector clocks的方法来比较同一个object不同版本之间的因果关系.vector clock的size会随着时间增长变得越来越大,怎么解决,设定一个值,当向量长度超过这个值,就把以前的丢掉.

###4.5 Execution of get() and put() operations###

刚才提到,发生错误时,可能会得到数据不一致,如何确保得到的是最新的版本呢,简单的来说,假设副本需要存在N个节点上,那么一个简单的要求就是最小的get()返回节点数R,加上最小的put()返回节点数W,大于总节点数N即可.即R+W>N.这样总能保证get()到的节点中,有最新的数据.而因为不需要get()和put()等所有节点返回,只需要R个或W个返回, 所以大大降低了系统延迟.

###4.6 Handling Failures: Hinted Handoff###

传统的quorum不够用啊,用"sloppy quorum"替代,什么意思?

简单的说,读写在first N个健康的节点上进行,但某1个或几个节点因为各种问题挂了,那么后续节点顶上,并把自己顶替谁记住(Hint in).当挂掉的节点恢复时,再把数据恢复回去.

###4.7 Handling permanent failures: Replica synchronization###

两节点的同一个object数据不一致怎么办,需要把新的数据覆盖旧的,怎么来做更有效率.采用**Merkle tree**的方法.做Hashing树,Hash值一样,不用管,Hash值不一样,往树的孩子找,找到最终不一样的叶子节点,然后sync.

Merkle Tree是由每个节点为不同的key分别建立的.

###4.8 Membership and Failure Detection###

主要提到一个gossip-based protocol的方法,来检测Membership,类似于传八卦的方法,有节点发现某个节点挂了,它随机选择几个节点告诉它们谁谁谁挂了,依次类推

###4.9 Adding/Removing Storage Nodes###

比较好理解,不看了.

后面性能评估的待有需要时再看,文虽好,但还是快读吐了.

---

[You can Download this paper **Here**.][1]

[1]: {{site.baseurl}}/papers/ds/amazon-dynamo-sosp2007.pdf
