---
layout: post
title: Paper Reading - Selective Symbolic Execution
catagories: [ReadingNotes]
tags: [Symbolic Execution,  Paper]

---

符号执行是做程序行为分析,漏洞发掘,测试样例生成的很强悍的工具,但它还存在某些限制:目前能拿做符号执行的最大的程序也顶多几千行而已.为确保符号执行的可行性,实际上程序必须剥离它与Libs,OS以及硬件的交互.这篇paper给出符号执行(selective symbolic execution),看起来像是full system的符号执行,其实只跑开发者关心的那一部分.我们设计了一个原型,可以符号执行一个完整系统的任何部分,包括应用,库,操作系统和硬件设备.它无缝的在具体执行和符号执行之间进行转换.我们的技术使符号执行在实际应用在真实环境上跑大型软件,而不需要对这些环境进行建模.

1. Introduction
----------------

符号执行做为自动化的软件测试方法,越来越火,尤其是在学习恶意代码行为时.用符号执行方法发现的行为特征(比如bugs)可以很简单的使用符号执行时收集到的信息再次重现,因此这种方法对开发者和测试者来说很强力,很划算.

当程序被符号化的执行时,它为每个输入提供符号化的值(e.g..a,r),来替换具体的正常输入.程序中的每个赋值操作,顺着给定的执行路径,使用符号化的表达式来更新程序变量,而不是一个具体的表达式计算.一个条件状态(if x>0 then .. else..),分成两条新的路径来执行,一条是then的路径,一条是else的分支,都要带一个共同的前缀.then的路径被if条件的(y-2>0)约束,同理else.约束值结合在一起,就形成一条路径,这条路径可以通过约束求解的办法来算出具体的输入(这个输入就使程序沿着这条路径来执行).比如到达assert()语句的路径,所有符号的约束求解构成的测试样例就可以使程序重现相应的崩溃场景.

符号执行引擎可以以某种引擎的数据结构,典型性的表现出程序的内存情况,并且当执行到达一个分支时,引擎就会fork当前的程序状态.这个每个路径,都有该程序的状态的私有版本.这样的话,引擎就可以让所有所有独立起来,并且并发的执行起来.

执行路的大小,随着条件数量的增加,而指数级的增长,这就带来**路径爆炸**问题.状态的总数和约束的数量随着符号执行的继续相应增长.各种优化措施被提出,但是目前符号执行引擎仍然只能承受几千行的小程序.因此这种方法还不能应用于我们今天所用到的大多数的工具软件(firefox, openoffice, mysql, eclipse等),它们中的每一个都上百万行的代码.

符号执行的另一个挑战是与环境的交互问题.比如当一个类似于firefox的程序读一个socket时,它调用libc,紧接着执行system call, 然后唤醒网卡设备,再从网卡读数据.而符号化的执行firefox,可能需要符号执行所有被调用的库,操作系统和驱动,但是这样加重路径爆炸问题,并且需要过多的内存空间和CPU时间.替代的解决办法是现有的工具自己来建立通用的库文件并且抽象执行环境,这样就可以把程序独立出来,保持符号执行在目标程序的范围内.但是建立一个完整的模型,很困难,劳力密集型的,因为库和操作系统有成千上万的语意复杂的API函数.这样的Models很少,所以符号执行只能被限制用在与外界交互很少的程序上.

我们注意到开发者一般不会测试整个系统,而是只关注很小一部分,一个内核模块,最近增加的功能,最近修改的数据结构相关的代码等等.因此,整个系统的符号执行是不必要的.

这篇文章提出选择符号执行的方法([S2E][1]),一种提供貌似整个系统符号执行的技术.我们的虚拟执行平台允许用户指定系统空间的一个特定范围,然后关注这一块符号化探测出来的的CPU和内存资源,看起来像整个系统在符号执行,其实这个范围之外是实际执行的.执行状态无缝的在符号执行和真实执行之间切换.

[S2E][1]的主要挑战是在确保一致性和高效执行的同时,让这种切换流透明化.

2. Use Cases
------------

[S2E][1]可以用在很多复杂的软件开发任务:

###Tesing in complex evironments

真实的程序,如Firefox, 使用多个Libs.用符号化的值来跑真实的程序,需要执行各种库,操作系统等等.现有符号执行引擎水平,可以做单向的符号数据到具体数据的转换,但它们不能处理返回的路径.传给read()一个buffer的指针,不可能将返回值当做符号来用.在[S2E][1]中,可以具体化感兴趣的一块程序,s2e会正确的,符号化的来执行它,而不管它到底有多少libs和kernel的调用.根据符号执行树,这个使用安全表明彻底的修剪call到环境的子树,可以明显的减少总的树大小.

###Fine-grain module testing

当测试例如Firefox之类的大程序时,开发者通常会关注某一特定区域的代码,比如死锁模块,或新增修改的代码.[S2E][1]允许指出可执行段的哪一块可以被符号执行,而其它的代码正常执行.在这种使用情况下,所有不与指定代码的相交的路径,都会从符号执行树中删掉.与单元测试方法比较,S2E提供更充足的灵活性,执行可以进入一个模块1次以上,可以跨越多个层(库,OS内核,驱动等).

###Data-driven testing

开发者经常变动数据结构,所以需要测试所有与这个改动的数据结构相关的代码,而不管这些代码是哪个模块.[S2E][1]可以选择性的符号执行这些代码,而不需要清楚的指定代码块.这个方法即使在指针混淆时还能工作,能确保所有想要的代码路径都能覆盖.在这种使用情况下,s2e完全删掉那些不会触及到这个数据结构的路径.

###Hybrid-input testing

另一种选择执行路径的方法是在程序输入的地方放约束条件.比如0<Y<99,可能编码成一列表示值.程序会符号化的执行这个约束.更严格的约束,比如y=1,使输入具体化,有一些输入可以被完全具体化,有一些混合,其他的需要完全符号化(初始化时没有限制).类似的测试比如基于语法的fuzz测试工具或混合符号执行引擎,但是它们需要程序用这些具体的值再跑一遍.[S2E][1]更有效率,因为它不需要这样.有时,一些bugs潜伏在执行树很深的地方,正常的符号执行引擎,需要很长的路径才能到达.而使用初始化约束的输入,s2e可以允许其它输入,比如网络包,把它们符号人,从而减少路径爆炸的问题:条件是进入具体的值,不会fork新的状态,因此将这些路径从树里删掉.

###Reproducing user-reported bugs

当用户归档漏洞报告时,他们努力提供更详尽的故障现象,输入和配置,来帮助开发者内部重现.这种问题重现,是调试时的关键性的第一步,但是可能会相当困难,尤其是在并行程序中.符号执行可以搜索到一条被证明是错误的路径,选择符号执行可以当查找的工作更具有效率.具体的输入和配置参数将那些在错误过程中不会发现的路径削减掉,而剩余的其余的输入都是符号化的.换句话来说,漏洞报告的细节明确了执行的包络,[S2E][1]就在这种包络范围内来搜索有问题的路径.

###Dynamic failure analysis

考虑到那些在虚拟机里运行的程序,周期性的checkpoint.如果程序崩溃掉,[S2E][1]可以从最近的checkpoint选择性的符号执行,来找到导致当前状态崩溃的路径,整个过程完全自动化.

###Failure avoidance

类似以上使用案例,[S2E][1]可以以现在状态之前,符号化的执行一小步,来判断将来的执行路径上有没有可能的错误.比如,程序可以使用s2e来判定如果我使用了一个特殊的锁,会不会有死锁现象发生,如果有,那就避免获取那个锁.

###Reverse-engineering programs

[S2E][1]可以以较高的覆盖率,选择性的符号执行一个闭源的二进制的设备驱动程序.如果我们搜集到硬件交互的痕迹,我们可以逆向出这个驱动的状态机.这种情况下s2e的主要挑战是驱动是与系统内核和硬件紧紧贴合在一起的.我们的s2e引擎是通过提供符号化的OS内核和硬件接口的输入,来符号化的执行设备驱动的.结果,驱动不知道具体运行的内核,内核不知道驱动是符号化执行的.s2e甚至可以模拟硬件,以免除真实驱动的需求.

3 Selective Symbolic Execution
-------------------------------

我们把一个系统看做一个大的程序.选择符号执行是一种可以指定哪些用来符号执行,哪来来具体的方式.这种选择,只跑那些必要的部分,这也是把符号执行扩展到真实系统上的一个关键因素.

code&data.在[S2E][1]中,通过执行文件名,目标文件,或内核或程序代码段的一个程序计数序列.然后s2e可以符号化执行这些代码,所有参与条件跳转的变量符号化,来确保这个代码段内所有的路径都能被探测到.还可以指定系统状态的一部分,通过指出一个数据结构或内核和程序段的一段地址.s2e然后符号化的标注这些数据,然后符号化的执行对那段数据进行读写的所有代码.其它的就按真实的来跑.

主要挑战:在确保正确性和效率的前提下,使用符号化和具体的数据在执行过程中共存.一般而言,我们可以将整个系统状态都看做符号化,通过改变每一个字节范围的约束数量.一个具体的变量就是一个约束值有个某个常量.指令操作的具体状态可以原样的跑,但是符号化的状态必须模拟.因此我们在具体域和符号域之间有一个严格的边界,执行过程穿越这个边界的时候,数据必须能适当的来回切换.[S2E][1]的贡献:在必要的状态转换之下,正确执行一个真实系统时,并且最大化的原生态执行.

一个例子来表明这种转化:测试一个闭源的二进制网络设备驱动程序,驱动的机器码符号化的执行,而应用,内核和固核具体执行.为了简化这种表示,假设我们的检查目标是否驱动到达导致崩溃的assert()状态.如果有,对应的驱动参数会被保存成一个测试样例.一个简单的系统可以干这个测试的活.

###Concrete-->Symbolic

当应用读数据时,它调用read(fd, buf,len)(libc中的函数),然后导致一个系统调用,而后调用网卡驱动,把DMA中的len长度的字节读到内核缓存中,然后被拷贝,让将用户态的缓存指针指向它.这需要驱动写特定的值到NIC硬件寄存器.应用调用的库函数read(),使用的是具体的参数,最终会传给内核的,并调用驱动中的drv_read()函数.

[S2E][1]可用来探测从drv_read()开始的所有的执行树.s2e把drv_read()函数的参数都符号化,然后来探测所有路径.或者,如果想使用特定的工作负荷来测试驱动的话,可以保持drv_read()的某些参数具体化.

drv_read()的参数,不只输入到驱动程序中,首先它们是硬件响应的,使用[S2E][1]来把它们符号化.第二,还有一些非数据的输入,比如时钟,硬件中断.这些据我们所知,以前都不在符号执行的考虑范围之内.一个符号化的中断,有个符号化的交付时间.也就是可以让驱动在任何点交付(不知何意),使遭受来自执行过程中的约束.

不考虑drv_read()的参数是否符号还是具体的,所有的其它输入是符号的.因此,带符号参数的符号执行可以探测以drv_read()为根的所有路径.反过来说,带具体参数的符号执行会探测出所有具体调用可能执行的路径,找到它的所有的成功和失败的路径.

注意到真实硬件的存在是不必要的,因为基于驱动的期望,返回结果都可以被[S2E][1]引擎来模拟.这类似于提供一个隐含的硬件模型,我们称之为符号硬件.当然,也可能有一些驱动不期望的硬件行为,这些在没有真实硬件的情况下,是不能被模拟的.符号中断需要简化一个通用的硬件模型,来嵌入到s2e引擎中去.这个模型包含了通用的一些行为,比如DMA setup.s2e在中断传递时,把它看成约束对应到等价的中断处理行为.符号中断是找出内核和驱动中的并发bugs的关键特征.

###Symbolic-->Concrete

两个挑战:首先符号区域(driver)必须返回一个具体的值到具体的域(kernel).其次,符号域可能会调用具体域,比如驱动就可能调用kmalloc来获取一个临时的缓冲区,或它将值存到驱动的寄存器上.所有这些都需要将符号值转到具体值.

在第一个案例中,唯一的一致性要求是提供具体的参数,返回值是可行的.这可能通过它除了符号执行以外具体执行驱动,并返回一个具体的值即可.因为符号执行树包括具体参数执行的一个父集,[S2E][1]可以理论上的对应于一个路径而返回一个值.这个路径通过强加具体的参数,它的输入满足所有的约束.使用额外的具体执行,但是允许结果在符号执行完之前返回,后面就可以在后台异步的进行.(这一段没看懂)

在第二个案例中,当符号域调用到具体域中,符号调用参数可以具体化,依上所述,通过选择一个具体的值来满足目前的约束.但是这个具体化必须对应一个条件:所有的后来的路径,都必须加上这个约束.

这些约束可以限制从具体域探测返回的路径数量.因此[S2E][1]记录哪些约束是具体化的,哪些是分支跳转的.如果,在一个后续的点上,一个具体的约束限制了路径的选择.然后s2e记住这个连接点.s2e使用约束求解来找到一个y的最小集合,这个集合允许探测所有跳过的路径.然后它使用这些值重新执行这些调用到具体域中,来探测剩下的路径,以满足路径约束为条件.

如果这些调用有一些侧面效应(kmalloc),它们可能搅乱具体域(比如run out of memory),所以系统状态应该在每个调用到具体之前进行fork.[S2E][1]对系统状态进行完全的控制,所以做到这样是可行的.但是,一个原生态的实现可能代价很大,所以我们使用按需转化和缓存模式.

###On-Demand Conversion

这其中存在很多情况,尽管执行是在符号和具体之间穿梭的,数据的转化是不需要的.比如,一个应用从socket读数据并将其写到另一个socket中.网络数据从网卡的具体域传送到驱动的符号域,然后到内核/libc/application的具体域,然后再从Libc,内核等回到符号域,然后再回到NIC.如果数据只是从一个缓冲区拷到另一个,但却从来 没有在任何控制流中起作用,那么就没理由让它也符号化.类似的,如果驱动通过调用kernel分配一个缓冲区,然后使用一个内核函数拷贝一些符号数据到这些缓冲区,然后使用这些数据,但不需要内核做任何事情,只是拷贝.这不需要对这些符号化的数据进行具体化.一个类似的情况:当一个复杂的数据结构从符号域与具体域之间传送的时候,但是这个数据结构只有一个field真正被读,这意味着只有这个涉及到的field需要做转换.

基于这些原因,[S2E][1]按需要进行转换.总体来说,每个内存字节和CPU寄存器都关联一个metadata,来表明这是否是符号化的,如果是,它包括相应的约束.当字节被拷贝到另一个地方时,相应的Metadata也会被拷贝.conversion只有在数据被作为条件分支的一部分或做算术运算时才被会进行.比如当数据值变化时.s2e必须记录这些metadata是从哪拷过来的,因为一个conversion必须传播到所有相关的内存字节.在这种方式下,具体执行的代码可以透明化的处理符号数据.

不必要的转换,可能会是一种浪费,按需转换就可以减少这种浪费.静态分析可以减少将来不必要的转换.

4. [S2E][1] Prototype for x86 Binaries
----------------------------------

我们目前开发出了[S2E][1]的引擎,可以对x86二进制进行选择符号执行.我们选择机器码层次的执行,为了更大的灵活性,包括,s2e可以在闭源的系统进行工作,比如商业软件和OS等.

我们的原型建立在QEMU虚拟机和KLEE符号执行引擎之上.QEMU使用动态二进制翻译来把guest机器的指令翻译成host的指令.它支持多种的客户机架构,包括x86,MIPS,ARM.我们的原型目前仅在x86上支持(读这篇论文时,已支持arm),但是可以直接向其它架构进行扩展.KLEE是一个高效的符号执行引擎(LLVM 字节码).像下面所说,这种结构让[S2E][1]对系统状态有一个完全的控制,包括硬件的接口.

我们写了一个可以动态的将x86翻译到llvm的虚拟机,可以运行在QEMU上,它是在llvm-qemu基础上开发的.我们改QEMU来选择那些需要符号执行的x86指令,然后把它们转换到LLVM,再传给klee.所有的其它指令不进行翻译.未来的可以的一个优化,即使是那些需要符号执行的指令,我们也只翻译那些符号化的寄存器和内存到llvm.符号执行代码在具体的符号运算上,原生态的运行.

使[S2E][1]可用于实际的关键元素是机器状态的共享表示,被用于与QEMU和KLEE结合.原始的QEMU虚拟机管理虚拟CPU的状态,VM的物理内存和VM设备.KLEE在同样的状态上工作,但使用不同的数据结构.我们改动QEMU,使其用于klee状态的存储,这样KLEE的符号域可以与具体域(QEMU)进行同步,而不需要拷贝.这种具体的状态现在可以在任何需要符号执行的时候进行fork.通过对vm的物理内存进行直接操作(而不是guest os的虚拟内存),s2e可以无缝的支持IPC和共享内存.zero-copy和直接物理内存表示,其s2e免去Bitscope.

我们使用分层的cache模式来加速内存的查找.这个符号执行与生俱来的状态forking导致机器状态树增长迅速.KLEE通过copy-on-write的方法,来减轻这种情况.这意味着一个机器状态的表示,可以包括指向父状态的指针.当层次变得越来越深时,就像全状态的符号执行,父指针链会变得非常长,使查找更耗时.在[S2E][1]中,每一次查找是通过一个到祖先的指针链完全成的.我们更新了现在的到父指针的指针,使其直接指向ancestor.更新后,所以后来的查找就会变短.

如上文所述,[S2E][1]可以在符号化的硬件上跑驱动.最初,我们每一个虚拟设备(NE2000网卡,USB设备等),来返回符号结果.尽管工作量非常小(30分钟一个设备),我们宏愿选择QEMU中一个通用的层,用来居中协调与硬件的交互.同样我们也在这一层做符号化中断.因此,这样的话,完全不需要硬件的参与.改动虚拟驱动还是有一些用处的,比如磁盘需要存储加强的版本的metadata,它来抓到字节是否符号化的.如果符号化,就包含住相应约束.当应用后续读这个数据,s2e可以适应的翻译这些元数据,再根据需要进行转换.

**Preliminary Results**: 本着逆向工程的目标,我们成功的使用[S2E][1]原型机在windows在符号化的执行了一些闭源的驱动设备.最小的一个,ne2000网卡驱动,调用37个不同的内核函数(自旋锁操作, 内存分配, I/O寄存器处理,时钟处理, NDIS管理, HAL调用等).对这些内核函数进行建模的话,可能会大费周章.通过benchmark测试,s2e也仅比未更改的QEMU慢1.7倍而已,因此表明s2e是可行的.

##5. Discussion

这篇论文并没有说一些与选择符号执行实现架构相关的事.另一个可做的设计实现,是在源代码层,现在,另一个方法是使用二进制插装来控制域的转换.当然,基础的机器状态表示方法可能与先前不同.

虽然在[S2E][1]中把应用跑在一个真实上的系统上,省去了大量环境的建模,但是这些模型是否可靠,也是个问题.比如,windows维护它自己的寄存器以一种压缩的形式.如果一个测试应用把符号数据写进寄存器,现在的s2e需要在这个数据上积累很多冗余的约束.如果,换个方法,我们使用一个寄存器模型,那么这些约束metadata就不需要了.

重复的调用一个函数(API),来符号化执行,可能导致大量的冗余探测,尤其是如果参数每次都是符号化的时候.[S2E][1]可以把每个函数的执行树,保存到磁盘上,第一没有副作用,第二,在需要的时候,再把它读回来继续执行即可.这种方法可以帮助加速路径约束的计算.

##6. Related Work

[S2E][1]扩展符号执行是建立在以前的工作的基础上的.其中一个是concolic testing, 它是具体的运行一个程序,同时搜集路径约束限制.这些约束用来发现可以使程序到达另一个路径.Hybird concolic testing把随机输入生成与符号执行结合起来,来提升代码覆盖率.

Symbolic Java Path Finder(SJPF)主要面前单元测试:它具体的执行程序直到到达目标单元的代码,此时再切换到符号执行.既然SJPF设计用来测试独立的单元,那么它不会追踪符号域与具体域的数据.

[S2E][1]可以看作把以前的工作一般化:我们主要面向把以前的功能都集成一个单独的框架内.另外,s2e提供了一些以前没有的功能,比如整个系统的符号执行,符号中断,环境的一般模型,等等.

混合执行模式-那些不涉及符号操作的原生态指令-首先出现在EXE和DART中,然后是Bitscope,我们使用它来做为优化策略.混合模式执行,它自身并不提供必要的转换和状态追踪.

最后,我们使用[S2E][1]的符号状态追踪和按需具体化,搞定了早期的环境建模的工作.

##7. Conclusion

这里存在一些连接的执行,它们某一段是符号执行,某一段是具体执行.每一个都是它们的优缺点.我们提出选择符号执行,在用户需求和效率执行之前做tradeoff.

[S2E][1]提供一种貌似全系统符号执行的方法,当符号执行被严格按需执行.s2e帮助扩展符号执行,使其可以分析程序行为,查找漏洞,为直实环境生成测试样例,而不需要清晰的对这些环境进行建模.

[S2E][1]提供了看似整个软件栈的执行,包括应用,库,系统内核和设备驱动,甚至固件.我们讨论了这些方法是否可行,并会把符号执行用在更大型的系统之前,从而把符号执行用在测试真实的,多用途的软件系统上.

---

[You can download this paper **HERE**][1]

[1]: {{site.baseurl}}/papers/se/selective_symbolic_ex.pdf
