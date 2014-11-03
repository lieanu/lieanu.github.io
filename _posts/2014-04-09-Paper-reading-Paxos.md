---
layout: post
title: Paper Reading - Paxos Made Simple
---

> *水平有限,文虽不长,但还是完全不知所云...*

The Paxos algorithm, when presented in plain English, is very simple.

> 完全就是嘲讽啊...

![](http://www.lamport.org/leslie.jpg)

1. Introduction
----------------

Paxos算法来实现分布式容错很难理解?或许是最初版本的是希腊语的原因吧.其实它非常简单明了.它的核心是consensus算法(synod,暂时不知道是什么).

2. The Consensus Algorithm
---------------------------

###2.1 The Problem###

假设一组进程,可以propose values. Consensus Algorithm可以确保proposed values里面的一个被选中.如果一个value已经被选中,然后进程就可以得知chosen value.consensus的安全需求如下所示:

* 只有proposed的value,才会被选中.
* proposed的value,只有一个会被选中.
* A process never learns that a value has been chosen unless it actually has been.

目标是确保一些建议值,最终会被选择,如果一旦一个value已经被选择,某个进程最终会获得这个value.

proposers, acceptors, learners.3类agents.单独一个进程可以是超过一个agent.但是agent和process的映射,这里不关心.

假设agents可以通过发送消息,互相之间通信.我们使用传统异步,非拜占庭模式:

* Agents可以任意速度运转,可以停掉,可以重启.但是除非某个挂掉重启的agent可以记住某些信息.否则solution不可行.
* 消息可以任意长度被传递,可以被复制,也可能丢掉.但不能被破坏.

###2.2 Choosing a Value###

最简单的方法:只有一个单独的acceptor agent. Proposer发送一个proposal给acceptor,到达后,选择第一个proposed value.虽然简单,但这个方法不那么美好,因为如果acceptor failure了,下一步的工作也别谈了.

换个方法.假如有多个acceptor agents. 一个proposer发送proposed value给一组acceptors.某个acceptor可以接收这些proposed value.一个足够大的acceptors组接收了,这个value就被看做chosen了.多大才叫足够大? 我们让一个集合包含大多数agents, 那么两个集合,必然有一个以上是相同的.

    P1. 一个acceptor必须接收它收到的第一个proposal.

这样又会带来一个问题: N个Values可能同时会被不同的proposers propose.这导致一种情况:每个acceptor都接收到一个value.但没有一个value被agents的大多数所接收.即使是两个proposed values,假如各被一半acceptors所接收,那么坏一个acceptor就不可以知道哪个value被选中了.

P1和被大多数acceptors接收才算选中的需求,表明一个acceptor必须允许接收一个以上的proposal.我们通过给每个proposal赋值来记录不同的proposals.因此,一个proposal由一个proposal数和一个value组成.为防止混淆,不同的Proposal应该有不同的proposal number.一个value被选中,只有当这个proposal被大多数acceptors所接收时.这种情况下,我们才说proposal被选中了.

我们可以允许多个proposal被选中,但必须保证所有被选中的proposal有一个相同的值.

    P2. 如果值为v的proposal被选中了,那么任何被选的higher-numbered的proposal,其值也为v.

numbers是有序的,条件P2保证了只有一个值被选中的安全属性.

为了被选中,一个proposal必须被以少一个以上的acceptor选中.所以:

    P2a. 如果值为v的proposal被选中,任何一个acceptor上接收的higher-numbered的proposal其值也为v.

因为通信是异步的,一个proposal可以被一个从未接收过任何proposal的accptor(c)所选中.假设一个新的proposer被唤醒了,发布了一个value不同的higher-numbered的proposal.P1要求c接收proposal,违反了P2a.

    P2b. 如果值为v的proposal被选中,那么任意一个proposer发出的
        higher-numbered的proposal,其值也为v.

因此一个proposal必须在被某个acceptor接收之前,被某个proposer发布出去.

如果satisfy P2b? 假设某个proposal的number为m,value为v.任意一个number为n>m的proposal,其value也为v.用归纳法证明.m..(n-1)的值都应该是v. 假如m被选中,那么必须有个集合C,由大多数acceptors组成,集合中每个acceptor都接收了m.

    C中的每个acceptor收到的m..(n-1)的proposals,其值都应该为v.

任一集合S,包括大多数acceptors,所以至少包含集合C中一个成员.

    P2c. 任意(v,n),如果proposal已经issue了,那么有一个集合S(大多数acceptors),
        a.没有acceptor收到任何比n小的Proposal. 
        b.最高Number,值为v的proposal,其number必小于n.(没理解对....?)

一个proposer想要issue一个proposal(n),必须知道highest-numbered proposal小于n的,已经被大多数acceptors接收的.知道已经接收的很简单,要预测将来要接收的就很难.proposer要求acceptor不要接收任何小于n的proposal.

1. 某个proposer选择了一个新的proposal(n),发送一个请求到acceptor集合中的每一位,要求回复如下:
a. 约定:不再接收小于n的proposal.
b. 最大数小于n的proposal,已经被接收了.当然,如果有的话.
2. 如果这个Proposer从acceptor集合,收到要求的回复,那么它可以发出这个(n,v).

请求收到后,Proposer发出一个proposal,给某些acceptors. **accept request.**

这是proposer的算法.acceptor怎么搞?acceptor收到两种request:一个是prepare request, 一个是accept request.acceptor忽略所有不危及安全的request.

    P1a. 某个acceptor可以接收一个num为n的proposal.
        如果它没有回复一个大于n的prepare request.

我们现在有了一个完整的算法:选中一个value,满足要求的安全属性(假设numbers是唯一的).最终的算法可做稍做优化.

假设一个acceptor收到一个prepare request(n),但是它已经回复了一个大于n的prepare request,因此约定不接收这个新的n.因此,acceptor不会回复这个prepare request.所以我们让accptor忽略这个prepare request.

在这种优化策略下,acceptor只需要记住highest-numbered proposal,和它已经回复的highest-numbered prepare request.因为P2c,acceptor必须在它挂掉重启后,还能记住这些信息.

把proposer和acceptor放到一起来看,算法操作有以下两个阶段:

    Phase 1. 
    a. 一个proposer选择了num为n的proposal,并发出了一个prepare request.
    b. 如果acceptor收到一个比它已经回复过的number大的prepare request(n),
        那么它就会给回复,承诺不再接收小于n的proposal,
        回复中并带有highest-numbered proposal.(if any)

    Phase 2.
    a. 一个proposer收到prepare request(n)的回复,然后它发一个accept 
        request(n,v),v是回复的highest-numbered proposal对应的值.
        if any,如果没有highest-numbered proposal,就自己给个值.
    b. 如果一个acceptor收到accept request(n),除非它已经回复过
        大于n的prepare request,不然它应该接收这个Proposal.

一个proposer可以产生多个proposals,只要每个都按算法的步骤来.它也可以在任意时刻丢掉一个Proposal.因此正确性是保障的,虽然丢掉后,request和response的时间可能会有点长).如果有proposer已经发出了更高数的proposal,那么老的就可以丢掉了.

###2.3 Learning a Chosen Value###

为了知道一个值被选中了,learner必须找到一个被大多数acceptors所接受的proposal.最显而易见的方法是每当acceptor收到一个proposal时,给所有的learners一个回复.这种办法可以让learners尽快找到一个被选中的value.但是这种方法需要每一个acceptor给每一个Learner回复.这是acceptor数与learners数的乘积.

非拜占庭故障假设,让一个Learner更容易的从其它learners那找出一个被接收的value.我们可以让acceptor回复不同的Learner,当一个值被选中时,它会依次通知另一个Learner.这种方法需要额外的一个循环来发现chosen value.可靠性也有点低,因为distinguished learner可能会挂掉.但是它的回复数,只是acceptor与learners数目的相加而已.

更通用的一种方法,acceptors可以把回复发给一组不同的learners.当然,集合越大,可靠性越高,但通信也更复杂.

因为消息丢失的问题,一个value可能被chosen了,但没有learner知道.某个acceptor可以把自己知道的告诉learner,但它不可能知道某个proposal是否被大多数acceptor所接受.

> *:::太绕了,已经晕了...*

###2.4 Progress###

容易构造出一个场景:两个proposers,每个都在处理一个proposal序列(number递增),暂时还没有被选中的.Proposer P完成Phase 1, proposal number为n1.另一个proposer Q,完成Phase 1, number为n2>n1.P的Phase 2中,proposal n1的accept request被忽略了,因为acceptor不接受比N2小的proposals.所以P从从n3开始继续搞phase 1.当然,同样影响q的Phase2.

为确保能继续干活,找个distinguished Proposer专门来issue proposal.如果它可以与大多数acceptor通信,并且使用一个Greater number,那么它成功iusse一个proposal.

如果足够的系统保障(proposer,acceptors, network),liveness可以通过选一个distinguished proposer来保证.

###2.5 The Implementation###

Paxos算法:每个process扮演proposer, acceptor, learner的角色.算法先一个Leader,担任distinguished proposer和distinguished learner的角色.Paxos consensus 算法就如上所示,requests和responses就作为ordinary messages发送.要有存储,用来防止挂掉时,信息还能存起来.acceptor在发回复之前,先存.

再就是确保两个Proposal不会有相同的Number号.每个proposer记住它issue过的最大Number的proposal到存储上.

3. Implementing a State Machine
--------------------------------

pass.

---

[You can download this paper **HERE**][1]

[1]: {{site.baseurl}}/papers/ds/paxos-simple.pdf
