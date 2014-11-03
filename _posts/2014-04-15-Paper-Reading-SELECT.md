---
layout: post
title: Paper Reading - SELECT - a formal system for testing and debugging programs by symbolic execution

---

> 本文写自1975年,距今39年.

##1 Introduction

SELECT实验系统:

1. 生成有用的测试数据.
2. 在路径输出部分,为每个程序变量提供简单的符号值.
3. 使用用户自定义的断言,验证路径的正确性.

测试数据生成的功能很有用,但不能做为total solution.把用户的交互作用,组合在测试数据生成的过程中,会更有希望吗?SELECT是这么干的,它允许用户在某些点,插入断言语句,来约束测试数据.

###A. Mechanical program verification and its shortcoming

生产可靠的软件,要付出的精力很多,即使在大量的测试和调试之后,仍会潜伏着一些bug.形式化验证(formally verifying)是个方法.所有的这些形式化验证的技术,包括一些经验软件验证机,利用程序意向形式化约定, 这些约定使用形式化断言语言来写的.验证程序然后在于分析程序的进行的动作,然后检查,每当输入数据达到输入断言的条件时,输出断言会被满足.每个系统的检查程序,都不相同,但所有的实验性的验证程序工作方式大都一样.

形式化的验证的方法,想处理实际的大程序,还差得很远(PS. 如今39年后,依然如此).我们先列出一些原因,然后讨论为什么如今一些障碍已经被搞定的情况下还是不行.形式化验证的方法,将来还是不能提供实际程序调试的工具.如今形式化验证实际程序的障碍有如下几点:

1. inventing intermediate assertions
2. 完整表示程序的逻辑.
3. 缺乏表达力足够的判定语言.
4. 技术限制(速度,足够大的存储).
5. 需要程序员干预太多.

1-3说的是形式化判定及其语言的问题.Elspas1974,Wegbreit1974, Katz-Manna1973就是来简化程序员的工作的.mechanical理论证明,在未来几年内可以做出来足够的速度和存储效率的验证技术.但是,程序干涉可能会一直是必要的.

除了上面说的几个问题,形式化验证还有很多缺点,不能被完全克服.

1. assertion是正确的,但是程序含有BUG.
2. 程序是对的,但assertion没写对.
3. 程序和assertion都有问题.
4. program和assertion都没问题,但是验证理论有问题.

验证系统应该能发现程序正在干的事和程序想要干的事之间的差异.即使程序员的intend很模糊,通过一个意料之外的输入来告诉程序员,程序有一些非预期的动作.因此重点是分析程序正在干什么,而不是程序员想要做什么.这么一个系统可以在构建系统的时,帮助程序员明确他模糊的路思,和事后分析一样.

因为编译器的实现不同,不同设备处理错误条件,溢出,除0和无穷运算的方法不同,所以无法保证验证结果能与想像的一样.所以在真实的环境里运行测试,还是一个很有用的补充手段.

###B. The purpose and general features of SELECT

Stanford Research Institute在1974年开始研究怎么绕过上面的这些缺点限制.利用形式化的方法,不确保数学上的正确性,不需要使用形式化断言.SELECT.构造输入约束来覆盖选择的路径,识别一些不可到达的路径,自动决定确切的输入数据来驱动程序测试.Howden, King(IBM).

##2. General Description of the SELECT Testing and Debugging System

###A. Overview of the System

主要特征:

1. 符号执行输入程序,覆盖从START到HALT的所有可能路径,如果有赋值操作,那么更新变量的符号值.symbolic predictaed是用来记录分支状态.
2. 布尔表达式,来表达执行的路径.
3. 根据输入符号值,简化程序变量的表达.
4. 生成真实输入数据,用于测试每个执行路径.
5. assertion,表达开发者的想法,或帮助系统选择测试数据,来满足约束条件.

###B. Path Analysis and Selection

path的定义,循环数不同,path不一样.无穷的循环搞不定(形式化验证能搞定,但不必要).符号执行只是来寻找问题,而不是数学上验证程序的正确性.循环太多,可以事先限定(应该有循环展开的方法).

概念上来说,路径选择和测试数据生成,应该是不同的进程来干这个事情.但是考虑效率问题,我把都它们在如今的SELECT版本里,合并在一起.

在讨论路径条件生成之前,还是先讨论一下程序语言的语法分析和语义分析.

###C. Input Language

LISP.理由就不说了,只是看他们的一种方法,应该到现在来说,就不实际了.

1. 赋值操作. 
2. 算术运算.
3. 状态控制.
4. 变量的作用域.
5. 布尔函数.
6. 函数调用子程序.

###D. Path Condition Generator

路径条件生成(约束生成)的目的是构造路径执行的布尔表达式.

两种方法:backward substitution, forward substitution.

forward substitution更自然.

###E. Inequalities Solver

###F. Automatic Traversal of Paths

限定Loop不能超过S次(感觉这种展开循环的方法不太好,ie的很多漏洞,都是需要迭代很多次才能触发).

###G. User Supplied Assertions as an Adjunct to the Program Code

##3 Example

##4 Relate work

> 真不是懒,确实很多太难懂了,还是用的LISP,不过那时候C应该刚出来,还没被太广泛的使用.
> 唉,自己给自己找了个借口,等哪天有需求,再回头看吧.

---

[You can download this paper **HERE**][1]

[1]: {{site.baseurl}}/papers/se/select75.pdf
