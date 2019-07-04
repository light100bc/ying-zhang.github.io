title: DDIA中文版上架了
category: yi
tags:
date: 2018-12-01
---
《设计数据密集型应用（影印版）Designing data-intensive applications》，中文版是《数据密集型应用系统设计》。
这本书9月份就由中国电力出版社出版了。因为我习惯在互动出版网浏览新书，而它没有跟中国电力出版社合作，所以最近才发现。


<!--more-->

---

> 在买影印版之前，我已经从PDF文件读了几章。影印版到手后，从头至尾通读了一遍。
> PDF很早就泄露出来了，一直关注何时中文版上架。寒假前还尝试联系几家出版社，看看能不能参与翻译。结果收到的答复是已经有出版社着手翻译了，但不知是哪家。
> 于是静候。又怕落到不靠谱的译者手里，所以想试着翻译一点，选了第9章，主要是因为这章偏理论，此外还有一段关于CAP理论（定理）的评价。译了三四天，进展太慢，到 “图 9-4” 就搁置（放弃）了。我是先把PDF转成纯文本，然后粘到 Google 翻译，之后再修订。Google 翻译的质量很高了，但有一些比较绕的句子，也不太容易修订通顺。还有就是一些名词，Google 翻译也能比较准确的翻译一些名词，当然还是有需要修订的。比如 Linearizability，会被翻译为 **线性化**，我则译为 **线性一致性**； Leader 都译为主节点；Quorum 是一个比较头疼的词，译成 **法定人数** 太长了，又没有找到两三个字的词。

> 收到中文版后，对照了一下第9章，总体来说翻译质量还可以吧，毕竟自己体会到翻译一本书的困难了。
> 下面是自己几个月前翻译的那点儿，感兴趣的读者可以比较一下。
> 另外放上 DDIA 的 [第9章原文](/doc/DDIA_ch9.pdf)

-----

此外，还想引用一下书中 Alan Kay 的一段话（接受美国Dr. Dobb 杂志采访，2012）
> 在我看来，计算机技术是一种流行文化。流行文化只在乎认同感和参与感，活在当下，从不关心合作、过去或未来。我认为大部分为金钱而编写代码的人也是如此。他们不了解他们的文化来自哪里。

-----

# 第9章 一致性和共识

> 是要错误地活着，还是正确地死去?
> —— Jay Kreps, 关于Kafka 和Jepsen的若干说明 (2013)

正如第8章所讨论的，分布式系统中可能会出现很多故障。处理这些故障最简单的方法是直接让整个服务失效，并向用户显示错误信息。如果无法接受这个解决方案，我们需要找到一些方法来容忍错误，即某些内部组件出现故障时仍能保持服务正常运行。

我们将在本章讨论一些示例算法和协议，来构建容错的分布式系统。我们将假定第8章中的所有故障都有可能发生: 数据包可能丢失、乱序、重复或在网络上延迟任意时间; 时钟是近似的; 节点可以在任意时刻停顿 (例如，发生了内存垃圾收集) 或崩溃。

构建容错系统最好的方法是找到一些可以提供若干有用保证（Guarantee）的通用抽象。一旦实现了这些抽象，就可以基于其上构建应用程序，获得相应的保证。这与我们在第7章中关于事务的思路相同: 通过使用事务，应用程序可以假装没有崩溃 (原子性)，没有其他人同时访问数据库 (隔离)，并且存储设备是绝对可靠的 (持久性)。即使发生了崩溃、竞态条件和磁盘故障，事务抽象也会屏蔽这些问题，应用程序无需操心。

现在我们沿着这个思路，寻求能够让应用程序忽略分布式系统中某些故障的抽象。例如，分布式系统中最重要的抽象之一是 **共识（Consensus）** ：即让所有的节点都达成一致意见。我们将在本章发现，在网络故障及进程故障的情况下，可靠地达成共识是非常棘手的。

一旦实现了共识算法，应用程序就可以将其用于多种目的。例如，对单主多副本的数据库，如果主节点宕机了，需要故障转移到另一个节点。其余节点可以用共识算法来选出新的主节点。正如第5章“处理节点宕机”(第156页)中讨论的，被所有节点都认可的主节点仅有一个是很重要的。如果有两个节点都分别认为自己是主节点，这种情况称为脑裂，经常导致数据丢失。正确实现的共识算法有助于避免此类问题。

在本章的“分布式事务和共识”小节（第352页），我们将研究解决共识问题的算法以及相关的问题。不过我们首先需要探究分布式系统中可以提供哪些抽象，以及何种程度的保证。

我们需要了解哪些事情是可行的，哪些是不可行的: 某些情况下，系统可以容忍故障并继续工作; 在其它情况下则是不可能的。理论和实践已经深入探索了可能和不可能的界限。我们将在本章概述这些根本限制。

这些主题已经在分布式系统领域研究了几十年，因此有很多资料 —— 我们只能介绍一点皮毛。本书无法详细介绍形式模型和证明的细节，我们将采用非正式的直观介绍。如果你感兴趣，参考文献提供了大量的深入内容。

## 一致性保证

在第5章的“副本滞后问题”小节（第161页），我们讨论了多副本数据库中发生的一些时序问题。如果在同一时刻查看两个数据库节点，那么你可能会在这两个节点看到不同的数据，因为写入请求到达这两个节点的时刻可能是不同的。无论数据库使用哪种复制方法（单个主节点，多个主节点或无主节点），不一致的情况都可能出现。

大多数多副本数据库至少都提供了 **最终一致性**（Eventual consistency）。这意味着，如果你停止数据库写入操作并等待一段不确定的时间，那么最终所有读请求都会返回相同的值[1]。换句话说，不一致是暂时的，最终会自行化解（假设网络中的任何故障最终都会修复）。将最终一致性称为 **收敛性** （Convergence）可能更合适，因为我们预料所有的副本最终会收敛到相同的值[2]。

然而，这是一个非常弱的保证 —— 它没有说 **什么时候** 副本将收敛。在收敛之前，读操作可能会返回任何值或空值[1]。例如，如果你写了一个值，然后立即再次读取，则不能保证你能看到刚写入的值，因为读取可能会被转发到不同的副本（请参阅第5章的“写后读Reading Your Own Writes”，第152页 ）。

最终一致性对于应用程序开发人员来说很难，因为它与普通单线程程序中变量的行为有很大不同。如果将值赋给一个变量，稍后再读取它，单线程的程序不会读到旧值或读取失败。表面上看数据库像是可以读写的变量，但事实上它的语义要复杂的多[3]。

使用仅提供较弱保证的数据库时，你需要时刻了解其局限性，而不是无意中做了太多假设。涉及的错误通常很微妙的，很难通过测试发现，因为应用程序在大多数情况下可能运行良好。当系统遇到故障（例如网络中断）或高并发时，最终一致性的特例才会显现出来。

在本章中，我们将探讨数据系统可能提供的更强的一致性模型。更强的一致性不是没有成本的：与较弱保证的系统相比，系统性能可能更差或容错性更低。尽管如此，更强大的保证可能更具吸引力，因为它们更容易正确使用。了解了几种不同的一致性模型后，你将能更好地决定哪一种最适合你的需求。

分布式一致性模型与我们先前讨论的事务隔离级别的层次结构有一些相似之处[4，5]（请参见第7章“弱隔离级别”，第233页）。虽然存在一些重叠，但它们关注的是不同的方面：事务隔离主要是为了避免由于并发执行事务而导致的竞态，而分布式一致性主要是在延迟和故障时协调副本的状态。

本章涵盖了范围广泛的主题，但我们将会看到，这些领域实际上是有深刻联系的：


+ 我们首先介绍通常用到的最强的一致性模型，线性一致性（Linearizability），并讨论其优缺点。
+ 然后，我们将讨论分布式系统中事件顺序的问题（“顺序保证”，第319页），特别是关于因果序和全序的问题。
+ 在第三部分（“分布式事务和共识”，第352页），我们将探讨如何原子的提交分布式事务，这将最终引导我们找到共识问题的解决方案。

## 线性一致性

在提供最终一致性的数据库中，如果你同时对两个不同的副本发起同一个查询，可能会得到两个不同的答案。这让人迷惑。如果数据库能够让人感觉只有一个副本（即只有一份数据），那么不就简单多了？这样每个客户端都会有相同的数据视图，并且不必担心副本滞后。

这就是 **线性一致性** （linearizability）[6]的想法（也称为 **原子一致性** （atomic consistency）[7]，**强一致性** （strong consistency），**即时一致性** （immediate consistency）或 **外部一致性** （external consistency）[8]）。线性一致性的准确定义涉及很多细节，我们将用这一小节来探讨。但基本的想法是让系统看起来好像只有一个数据副本，并且其上的所有操作都是原子的。有了这个保证，尽管实际上可能有多个副本，应用程序也不需要操心它们。

在可线性化的系统中，只要客户端成功完成写入，所有从数据库进行读取的客户端必须能够看到刚写入的值。保持数据单一副本的假象意味着保证读取到的值是最新的值，不是来自陈旧的缓存或副本。换句话说，线性一致性是对 **新近性** （recency）的保证。为了说明情况，我们来看一个非线性一致系统的例子。

<图 9-1.> 非线性一致的系统，导致了球迷的困惑

图9-1显示了一个非线性一致的体育网站的例子[9]。Alice和Bob正坐在同一个房间里，他们都在各自的手机查看2014年FIFA世界杯决赛的结果。在最后比分公布后，Alice刷新页面，看到已经宣布了获胜队，并兴奋地告诉Bob。Bob难以置信地刷新自己的手机，但是他的请求被发送到一个滞后的数据库副本，结果他的手机显示比赛还没结束。

如果Alice和Bob在同一时刻刷新页面，但获得了两个不同的查询结果，那么还不那么令人惊讶，因为他们不确定服务器先处理的哪个请求。然而，Bob明明知道他是在听到Alice对最后比分的欢呼之后才点击了刷新按钮（发起了他的查询），因此他期望他的查询结果至少与Alice一样新。他的查询返回旧结果的这个事实违反了线性一致性。

### 什么使系统可线性化？ What Makes a System Linearizable? 

线性一致性背后的基本思想很简单：使系统看起来好像只有一个数据副本。然而，需要仔细考虑才能精确地说明其涵义。为了更好地理解线性一致性，让我们多看几个例子。

图9-2显示了三个客户端同时在可线性化数据库中读写相同的键 $$x$$。在分布式系统论文中，$$x$$ 被称为寄存器（register） —— 实际上，它可以是键值存储中的一个键，关系数据库中的一行或文档数据库中的一个文档。

<图 9-2.> 如果读取请求与写入请求是并发的，那么旧值或新值都可能被返回。

简单起见，图9-2仅显示了来自客户端的请求，不考虑数据库。每段横条都是客户端发出的请求，首端是发出请求的时间，末端是客户端收到响应的时间。由于不定的网络延迟，客户端无法确定数据库何时处理其请求 —— 只知道必定是从发出请求到接收响应这段时间之内处理的。（脚注：图中一个微妙的细节是，假定存在着一个以水平轴表示的全局时钟。尽管真实系统通常没有准确的时钟（请参阅第8章“不可靠的时钟”，第287页），但这种假设是可行的：为了分析分布式算法，我们可以假设存在精确的全局时钟，只要算法不使用它即可[47]。算法只能看到由石英晶振和NTP产生的近似值。）

在这个例子中，寄存器有两种类型的操作：

 

+ $$read(x) ⇒ v$$ 表示客户端请求读取寄存器 $$x$$ 的值，数据库返回值 $$v$$。
+ $$write(x, v) ⇒ r$$ 表示客户端请求将寄存器 $$x$$ 赋值为 $$v$$，数据库返回响应 $$r$$（可以是正常或错误）。


在图9-2中，$$x$$ 的初始值为 0，客户端C向其写入1。在写入过程中，客户端A和B反复轮询数据库以读取最新值。A和B可能读到哪些值呢？

+ 客户端A的第一个读取操作在C写入开始之前就完成了，因此必须返回旧值0。
+ 客户端A最后的读操作在C写入完成后才开始，所以如果数据库是线性一致的，它肯定必须返回新的值1：我们知道具体的写入必然是在对应请求的开始与结束之间的某个时刻进行处理的，同样具体的读取必然是在对应请求的开始与结束之间的某个时刻进行处理。如果在写入结束后才开始读取，则读取的处理必然晚于写入，因此必须要读到已经写入的新值。
+ 与写操作有时间重叠的读操作可能会返回0或1，因为我们不知道写操作在处理读操作时是否已经生效。读和写是并发的。

但是，这还不足以完全描述线性一致性：如果与写操作并发的读操作可能会返回旧值或新值，那么在写操作进行过程中，读进程也可能会看到这个值在旧值和新值之间反复变动。这不是我们所期望的所谓等效于“单一数据副本”的系统。（脚注：与写操作并发读取时可能返回旧值或新值的寄存器称为 **常规寄存器** （regular register）[7, 25]。）

为了使系统可线性化，我们需要添加另一项约束，如图9-3所示。

<图 9-3.> 在任一次读取到新值后，任意客户端的后续所有读取也必须返回该新值。

在一个可线性化的系统中，我们可以想象，在写入操作开始与结束之间，必定存在某个时刻， $$x$$ 的值原子的从 0 变成 1。因此，如果一个客户端的读取返回新值1，即使写入操作尚未结束（响应尚未返回到对应的客户端），所有后续的读取也必须返回新值。

图9-3中用箭头表示这种时序依赖。客户端A是第一个读到新值1的。在A的读取返回之后，B开始新的读取。由于B的读取严格在A读取之后发生，因此即使C的写入仍在进行中，数据库也必须返回1。（与图9-1中Alice和Bob的情况相同：在Alice读取新值之后，Bob也希望读取到新值 。）

我们可以进一步细化这个时序图，显示出每个操作在某个时刻以原子的方式生效。图9-4是一个更复杂的例子[10]。

在图9-4中，除了读写之外，我们还添加了第三种类型的操作：

+ $$CAS(x，v_{old}，v_{new})⇒ r$$ 表示客户端请求原子的 **比较并置位** 操作（请参阅第7章“比较并置位”，第245页）。如果寄存器 $$x$$ 的当前值等于 $$v_{old}$$，则原子地将其值置为 $$v_{new}$$。如果 $$x ≠ v_{old}$$，则不改变寄存器的值，并返回错误。 $$r$$ 是数据库的响应（正常或错误）。

图9-4中的每个操作都用横条内的竖线标记出来了，对应着我们认为操作发生的时刻。这些标记可以顺序排列起来，而且结果必须是一个有效的寄存器读写序列（即每次读取都必须返回最近写入的值）。

线性一致性的要求是，操作标记排成的队列总是向前移动（从左到右），从不后退。这一要求确保了我们前面提到的新近性的保证：一旦写入或读取了新值，后续都会读到写入的值，直到它再次被覆盖。

<图 9-4.> 标出读取和写入生效的时刻。B最后的那次读取的是非线性一致的。
...

### 应用线性一致性 Relying on Linearizability
...

#### 分布式锁和主节点选举 Locking and leader election
...

#### 约束和唯一性保证 Constraints and uniqueness guarantees
...

#### 跨信道的时序依赖 Cross-channel timing dependencies
...

### 实现线性一致的系统 Implementing Linearizable Systems
...

#### 线性一致性和 Quorums Linearizability and quorums
...

### 线性一致性的成本 The Cost of Linearizability
...

#### CAP理论 The CAP theorem

...
> 以下是 Google 翻译的内容（https://translate.google.cn/ 没有被墙，可以正常访问）


**无益的CAP定理** 

CAP有时表现为一致性，可用性，分区容差：从2中挑选2个。不幸的是，这样做是误导[32]因为网络分区是一种错误，所以它们不是你可以选择的东西：无论你喜欢与否，它们都会发生[38]。
在网络正常工作时，系统可以提供一致性（线性化）和总体可用性。发生网络故障时，您必须在线性化或总可用性之间进行选择。因此，一种更好的表达CAP的方法是在分区时保持一致或可用[39]。一个更可靠的网络需要不经常做出这种选择，但在某些时候，选择是不可避免的。
在CAP的讨论中，对可用性一词有几个相互矛盾的定义，而作为定理的形式化[30]与其通常的含义不相符[40]。许多所谓的“高可用性”（容错）系统实际上不符合CAP对可用性的特殊定义。总而言之，CAP周围存在很多误解和混淆，它无助于我们更好地理解系统，因此最好避免使用CAP。

正式定义的CAP定理[30]范围非常狭窄：它只考虑一种一致性模型（即线性化）和一种故障（网络分区，或者活着但彼此断开的节点）。它没有说明网络延迟，死节点或其他权衡。因此，虽然CAP在历史上具有影响力，但它对设计系统几乎没有实际价值[9,40]。
在分布式系统中有更多有趣的不可能性结果[41]，CAP现在已被更精确的结果所取代[2,42]，因此它在今天主要是历史兴趣。


> 原文


**The Unhelpful CAP Theorem**

CAP is sometimes presented as Consistency, Availability, Partition tolerance: pick 2 out of 3. Unfortunately, putting it this way is misleading[32] because network partitions are a kind of fault, so they aren’t something about which you have a choice: they will happen whether you like it or not [38].
At times when the network is working correctly, a system can provide both consistency (linearizability) and total availability. When a network fault occurs, you have to choose between either linearizability or total availability. Thus, a better way of phrasing CAP would be either Consistent or Available when Partitioned [39]. A more reliable network needs to make this choice less often, but at some point the choice is inevitable.
In discussions of CAP there are several contradictory definitions of the term availability, and the formalization as a theorem [30] does not match its usual meaning [40]. Many so-called "highly available" (fault-tolerant) systems actually do not meet CAP’s idiosyncratic definition of availability. All in all, there is a lot of misunderstanding and confusion around CAP, and it does not help us understand systems better, so CAP is best avoided.

The CAP theorem as formally defined [30] is of very narrow scope: it only considers one consistency model (namely linearizability) and one kind of fault (network partitions, or nodes that are alive but disconnected from each other). It doesn’t say anything about network delays, dead nodes, or other trade-offs. Thus, although CAP has been historically influential, it has little practical value for designing systems [9, 40].
There are many more interesting impossibility results in distributed systems [41], and CAP has now been superseded by more precise results [2, 42], so it is of mostly historical interest today.


#### 线性一致性和网络延迟 Linearizability and network delays
...
{% math %}{% endmath %} 

# 参考文献
     
[1]. Peter Bailis and Ali Ghodsi: **["Eventual Consistency Today: Limitations, Extensions, and Beyond"](http://queue.acm.org/detail.cfm?id=2462076)**, *ACM Queue*, vol. 11 no. 3, pp. 55-63 March 2013. **[doi:10.1145/2460276.2462076](http://dx.doi.org/10.1145/2460276.2462076)** 
[2]. Prince Mahajan, Lorenzo Alvisi, and Mike Dahlin: **["Consistency, Availability, and Convergence"](http://apps.cs.utexas.edu/tech_reports/reports/tr/TR-2036.pdf)**, *University of Texas at Austin, Department of Computer Science*, Tech Report UTCS TR-11-22, May 2011.
[3]. Alex Scotti: **["Adventures in Building Your Own Database"](http://www.slideshare.net/AlexScotti1/allyourbase-55212398)**, at *All Your Base*, November 2015.
[4]. Peter Bailis, Aaron Davidson, Alan Fekete, et al.: **["Highly Available Transactions: Virtues and Limitations"](www.vldb.org/pvldb/vol7/p181-bailis.pdf)**, at *40th International Conference on Very Large Data Bases (VLDB)*, September 2014. Extended version published as **[arXiv:1302.0309 [cs.DB]](http://arxiv.org/pdf/1302.0309.pdf)**.
[5]. Paolo Viotti and Marko Vukolić: **["Consistency in Non-Transactional Distributed Storage Systems"](http://arxiv.org/abs/1512.00168)**, arXiv:1512.00168, 12 April 2016.
[6]. Maurice P. Herlihy and Jeannette M. Wing: **["Linearizability: A Correctness Condition for Concurrent Objects"](http://cs.brown.edu/%7Emph/HerlihyW90/p463-herlihy.pdf)**, *ACM Transactions on Programming Languages and Systems (TOPLAS)*, vol. 12, no. 3, pp. 463–492, July 1990. **[doi:10.1145/78969.78972](http://dx.doi.org/10.1145/78969.78972)** 
[7]. Leslie Lamport: **["On interprocess communication"](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/12/On-Interprocess-Communication.pdf)**, *Distributed Computing*, vol. 1, no. 2, pp. 77–101, June 1986. **[doi:10.1007/BF01786228](http://dx.doi.org/10.1007/BF01786228)** 
[8]. David K. Gifford: **["Information Storage in a Decentralized Computer System"](http://www.mirrorservice.org/sites/www.bitsavers.org/pdf/xerox/parc/techReports/CSL-81-8_Information_Storage_in_a_Decentralized_Computer_System.pdf)**, Xerox Palo Alto Research Centers, CSL-81-8, June 1981.
[9]. Martin Kleppmann: **["Please Stop Calling Databases CP or AP"](http://martin.kleppmann.com/2015/05/11/please-stop-calling-databases-cp-or-ap.html)**, *martin.kleppmann.com*, May 11, 2015.
[10]. Kyle Kingsbury: **["Call Me Maybe: MongoDB Stale Reads"](https://aphyr.com/posts/322-call-me-maybe-mongodb-stale-reads)**, *aphyr.com*, April 20, 2015.
[11]. Kyle Kingsbury: **["Computational Techniques in Knossos"](https://aphyr.com/posts/314-computational-techniques-in-knossos)**, *aphyr.com*, May 17, 2014.
[12]. Peter Bailis: **["Linearizability Versus Serializability"](www.bailis.org/blog/linearizability-versus-serializability/)**, *bailis.org*, September 24, 2014.
[13]. Philip A. Bernstein, Vassos Hadzilacos, and Nathan Goodman: **["Concurrency Control and Recovery in Database Systems"](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/05/ccontrol.zip)**, Addison-Wesley, 1987. ISBN: 978-0-201-10715-9, available online at research.microsoft.com.
[14]. Mike Burrows: **["The Chubby Lock Service for Loosely-Coupled Distributed Systems"](http://research.google.com/archive/chubby.html)**, at *7th USENIX Symposium on Operating System Design and Implementation (OSDI)*, November 2006.
[15]. Flavio P. Junqueira and Benjamin Reed: *"ZooKeeper: Distributed Process Coordination"*, O'Reilly Media, 2013. ISBN: 978-1-449-36130-3
[16]. *"etcd 2.0.12 Documentation"*, CoreOS, Inc., 2015.
[17]. **["Apache Curator"](curator.apache.org)**, Apache Software Foundation, *curator.apache.org*, 2015.
[18]. Morali Vallath: *"Oracle 10g RAC Grid, Services &amp; Clustering"*, Elsevier Digital Press, 2006. ISBN: 978-1-555-58321-7
[19]. Peter Bailis, Alan Fekete, Michael J Franklin et al.: **["Coordination-Avoiding Database Systems"](http://arxiv.org/pdf/1402.2237.pdf)**, *Proceedings of the VLDB Endowment*, vol. 8, no. 3, pp. 185–196, November 2014 also at arxiv:1402.2237.
[20]. Kyle Kingsbury: **["Call Me Maybe: etcd and Consul"](https://aphyr.com/posts/316-call-me-maybe-etcd-and-consul)**, *aphyr.com*, June 9, 2014.
[21]. Flavio P. Junqueira, Benjamin C. Reed, and Marco Serafini: **["Zab: High- Performance Broadcast for Primary-Backup Systems"](https://pdfs.semanticscholar.org/b02c/6b00bd5dbdbd951fddb00b906c82fa80f0b3.pdf)**, at *41st IEEE International Conference on Dependable Systems and Networks (DSN)*, June 2011. **[doi:10.1109/DSN.2011.5958223](http://dx.doi.org/10.1109/DSN.2011.5958223)** 
[22]. Diego Ongaro and John K. Ousterhout: **["In Search of an Understandable Consensus Algorithm (Extended Version)"](http://ramcloud.stanford.edu/raft.pdf)**, at *USENIX Annual Technical Conference (ATC)*, June 2014.
[23]. Hagit Attiya, Amotz Bar-Noy, and Danny Dolev: **["Sharing Memory Robustly in Message-Passing Systems"](http://www.cse.huji.ac.il/course/2004/dist/p124-attiya.pdf)**, *Journal of the ACM*, vol. 42, no. 1, pp. 124 – 142, January 1995. **[doi:10.1145/200836.200869](http://dx.doi.org/10.1145/200836.200869)** 
[24]. Nancy Lynch and Alex Shvartsman: **["Robust Emulation of Shared Memory Using Dynamic Quorum-Acknowledged Broadcasts"](http://groups.csail.mit.edu/tds/papers/Lynch/FTCS97.pdf)**, at *27th Annual International Symposium on Fault-Tolerant Computing (FTCS)*, June 1997. **[doi:10.1109/FTCS.1997.614100](http://dx.doi.org/10.1109/FTCS.1997.614100)** 
[25]. Christian Cachin, Rachid Guerraoui, and Luís Rodrigues: *"Introduction to Reliable and Secure Distributed Programming, 2nd edition"*, Springer, 2011. ISBN: 978-3-642-15259-7,  **[doi:10.1007/978-3-642-15260-3](http://dx.doi.org/10.1007/978-3-642-15260-3)** 
[26]. Sam Elliott, Mark Allen, and Martin Kleppmann: **["Personal Communication"](https://twitter.com/lenary/status/654761711933648896)**, thread on *twitter.com*, October 15, 2015.
[27]. Niklas Ekström, Mikhail Panchenko, and Jonathan Ellis: **["Possible Issue with Read Repair?"](http://mail-archives.apache.org/mod_mbox/cassandra-dev/201210.mbox/%3CFA480D1DC3964E2C8B0A14E0880094C9%40Robotech%3E)**, email thread on *cassandra-dev* mailing list, October 2012.
[28]. Maurice P. Herlihy: **["Wait-Free Synchronization"](https://cs.brown.edu/%7Emph/Herlihy91/p124-herlihy.pdf)**, *ACM Transactions on Programming Languages and Systems (TOPLAS)*, vol. 13, no. 1, pp. 124–149, January 1991. **[doi:10.1145/114005.102808](http://dx.doi.org/10.1145/114005.102808)** 
[29]. Armando Fox and Eric A. Brewer: **["Harvest, Yield, and Scalable Tolerant Systems"](http://radlab.cs.berkeley.edu/people/fox/static/pubs/pdf/c18.pdf)**, at *7th Workshop on Hot Topics in Operating Systems (HotOS)*, March 1999. **[doi:10.1109/HOTOS.1999.798396](http://dx.doi.org/10.1109/HOTOS.1999.798396)** 
[30]. Seth Gilbert and Nancy Lynch: **["Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services"](http://www.comp.nus.edu.sg/%7Egilbert/pubs/BrewersConjecture-SigAct.pdf)**, *ACM SIGACT News*, vol. 33, no. 2, pp. 51–59, June 2002. **[doi:10.1145/564585.564601](http://dx.doi.org/10.1145/564585.564601)** 
[31]. Seth Gilbert and Nancy Lynch: *"Perspectives on the CAP Theorem"*, *IEEE Computer Magazine*, vol. 45, no. 2, pp. 30–36, February 2012. **[doi:10.1109/MC.2011.389](http://dx.doi.org/10.1109/MC.2011.389)** 
[32]. Eric A. Brewer: **["CAP Twelve Years Later: How the 'Rules' Have Changed"](http://cs609.cs.ua.edu/CAP12.pdf)**, *IEEE Computer Magazine*, vol. 45, no. 2, pp. 23–29, February 2012. **[doi:10.1109/MC.2012.37](http://dx.doi.org/10.1109/MC.2012.37)** 
[33]. Susan B. Davidson, Hector Garcia-Molina, and Dale Skeen: **["Consistency in Partitioned Networks"](http://delab.csd.auth.gr/%7Edimitris/courses/mpc_fall05/papers/invalidation/acm_csur85_partitioned_network_consistency.pdf)**, *ACM Computing Surveys*, vol. 17, no. 3, pp. 341–370, September 1985. **[doi:10.1145/5505.5508](http://dx.doi.org/10.1145/5505.5508)** 
[34]. Paul R. Johnson and Robert H. Thomas: **["RFC 677: The Maintenance of Duplicate Databases"](https://tools.ietf.org/html/rfc677)**, Network Working Group, January 27, 1975.
[35]. Bruce G. Lindsay, Patricia Griffiths Selinger, C. Galtieri, et al.: **["Notes on Distributed Databases"](http://domino.research.ibm.com/library/cyberdig.nsf/papers/A776EC17FC2FCE73852579F100578964/%24File/RJ2571.pdf)**, IBM Research, Research Report RJ2571(33471), July 1979.
[36]. Michael J. Fischer and Alan Michael: **["Sacrificing Serializability to Attain High Availability of Data in an Unreliable Network"](http://www.cs.ucsb.edu/%7Eagrawal/spring2011/ugrad/p70-fischer.pdf)**, at *1st ACM Symposium on Principles of Database Systems (PODS)*, March 1982. **[doi:10.1145/588111.588124](http://dx.doi.org/10.1145/588111.588124)** 
[37]. Eric A. Brewer: **["NoSQL: Past, Present, Future"](https://www.infoq.com/presentations/NoSQL-History)**, at *QCon San Francisco*, November 2012.
[38]. Henry Robinson: **["CAP Confusion: Problems with 'Partition Tolerance'"](http://blog.cloudera.com/blog/2010/04/cap-confusion-problems-with-partition-tolerance/)**, *blog.cloudera.com*, April 26, 2010.
[39]. Adrian Cockcroft: **["Migrating to Microservices"](http://www.infoq.com/presentations/migration-cloud-native)**, at *QCon London*, March 2014.
[40]. Martin Kleppmann: **["A Critique of the CAP Theorem"](http://arxiv.org/pdf/1509.05393.pdf)**, arXiv:1509.05393, September 17, 2015.
[41]. Nancy A. Lynch: **["A Hundred Impossibility Proofs for Distributed Computing"](http://groups.csail.mit.edu/tds/papers/Lynch/podc89.pdf)**, at *8th ACM Symposium on Principles of Distributed Computing (PODC)*, August 1989. **[doi:10.1145/72981.72982](http://dx.doi.org/10.1145/72981.72982)** 
[42]. Hagit Attiya, Faith Ellen, and Adam Morrison: **["Limitations of Highly-Available Eventually-Consistent Data Stores"](http://www.cs.technion.ac.il/people/mad/online-publications/podc2015-replds.pdf)**, at *ACM Symposium on Principles of Distributed Computing (PODC)*, July 2015. **[doi:10.1145/2767386.2767419](http://dx.doi.org/10.1145/2767386.2767419)** 
[43]. Peter Sewell, Susmit Sarkar, Scott Owens, et al.: **["x86-TSO: A Rigorous and Usable Programmer's Model for x86 Multiprocessors"](http://www.cl.cam.ac.uk/%7Epes20/weakmemory/cacm.pdf)**, *Communications of the ACM*, vol. 53, no. 7, pp. 89–97, July 2010. **[doi:10.1145/1785414.1785443](http://dx.doi.org/10.1145/1785414.1785443)** 
[44]. Martin Thompson: **["Memory Barriers/Fences mechanical"](https://mechanical-sympathy.blogspot.com/2011/07/memory-barriersfences.html)**, *sympathy.blogspot.co.uk*, July 24, 2011.
[45]. Ulrich Drepper: **["What Every Programmer Should Know About Memory"](http://www.akkadia.org/drepper/cpumemory.pdf)**, *akkadia.org*, November 21, 2007.
[46]. Daniel J. Abadi: **["Consistency Tradeoffs in Modern Distributed Database System Design"](http://cs-www.cs.yale.edu/homes/dna/papers/abadi-pacelc.pdf)**, *IEEE Computer Magazine*, vol. 45, no. 2, pp. 37–42, February 2012. **[doi:10.1109/MC.2012.33](http://dx.doi.org/10.1109/MC.2012.33)** 
[47]. Hagit Attiya and Jennifer L. Welch: **["Sequential Consistency Versus Linearizability"](http://courses.csail.mit.edu/6.852/01/papers/p91-attiya.pdf)**, *ACM Transactions on Computer Systems (TOCS)*, vol. 12, no. 2, pp. 91–122, May 1994. **[doi:10.1145/176575.176576](http://dx.doi.org/10.1145/176575.176576)** 
[48]. Mustaque Ahamad, Gil Neiger, James E. Burns, et al.: **["Causal Memory: Definitions, Implementation, and Programming"](http://www-i2.informatik.rwth-aachen.de/i2/fileadmin/user_upload/documents/Seminar_MCMM11/Causal_memory_1996.pdf)**, *Distributed Computing*, vol. 9, no. 1, pp. 37–49, March 1995. **[doi:10.1007/BF01784241](http://dx.doi.org/10.1007/BF01784241)** 
[49]. Wyatt Lloyd, Michael J. Freedman, Michael Kaminsky, and David G. Andersen: **["Stronger Semantics for Low-Latency Geo-Replicated Storage"](https://www.usenix.org/system/files/conference/nsdi13/nsdi13-final149.pdf)**, at *10th USENIX Symposium on Networked Systems Design and Implementation (NSDI)*, April 2013.
[50]. Marek Zawirski, Annette Bieniusa, Valter Balegas, et al.: **["SwiftCloud: Fault- Tolerant Geo-Replication Integrated All the Way to the Client Machine"](http://arxiv.org/abs/1310.3107)**, INRIA Research Report 8347, August 2013.
[51]. Peter Bailis, Ali Ghodsi, Joseph M Hellerstein and Ion Stoica: **["Bolt-on Causal Consistency"](http://db.cs.berkeley.edu/papers/sigmod13-bolton.pdf)**, at *ACM International Conference on Management of Data (SIGMOD)*, June 2013.
[52]. Philippe Ajoux, Nathan Bronson, Sanjeev Kumar, et al.: **["Challenges to Adopting Stronger Consistency at Scale"](https://www.usenix.org/system/files/conference/hotos15/hotos15-paper-ajoux.pdf)**, at *15th USENIX Workshop on Hot Topics in Operating Systems (HotOS)*, May 2015.
[53]. Peter Bailis: **["Causality Is Expensive (and What to Do About It)"](http://www.bailis.org/blog/causality-is-expensive-and-what-to-do-about-it/)**, *bailis.org*, February 5, 2014.
[54]. Ricardo Gonçalves, Paulo Sérgio Almeida, Carlos Baquero, and Victor Fonte: **["Concise Server-Wide Causality Management for Eventually Consistent Data Stores"](http://haslab.uminho.pt/tome/files/global_logical_clocks.pdf)**, at *15th IFIP International Conference on Distributed Applications and Interoperable Systems (DAIS)*, June 2015. **[doi:10.1007/978-3-319-19129-4_6](http://dx.doi.org/10.1007/978-3-319-19129-4_6)** 
[55]. Rob Conery: **["A Better ID Generator for PostgreSQL"](http://rob.conery.io/2014/05/29/a-better-id-generator-for-postgresql/)**, *rob.conery.io*, May 29, 2014.
[56]. Leslie Lamport: **["Time, Clocks, and the Ordering of Events in a Distributed System"](http://research.microsoft.com/en-US/um/people/Lamport/pubs/time-clocks.pdf)**, *Communications of the ACM*, vol. 21, no. 7, pp. 558–565, July 1978. **[doi:10.1145/359545.359563](http://dx.doi.org/10.1145/359545.359563)** 
[57]. Xavier Défago, André Schiper, and Péter Urbán: **["Total Order Broadcast and Multicast Algorithms: Taxonomy and Survey"](https://dspace.jaist.ac.jp/dspace/bitstream/10119/4883/1/defago_et_al.pdf)**, *ACM Computing Surveys*, vol. 36, no. 4, pp. 372–421, December 2004. **[doi:10.1145/1041680.1041682](http://dx.doi.org/10.1145/1041680.1041682)** 
[58]. Hagit Attiya and Jennifer Welch: *"Distributed Computing: Fundamentals, Simulations and Advanced Topics, 2nd edition"*, John Wiley &amp; Sons, 2004. ISBN: 978-0-471-45324-6,  **[doi:10.1002/0471478210](http://dx.doi.org/10.1002/0471478210)** 
[59]. Mahesh Balakrishnan, Dahlia Malkhi, Vijayan Prabhakaran, et al.: **["CORFU: A Shared Log Design for Flash Clusters"](https://www.usenix.org/system/files/conference/nsdi12/nsdi12-final30.pdf)**, at *9th USENIX Symposium on Networked Systems Design and Implementation (NSDI)*, April 2012.
[60]. Fred B. Schneider: **["Implementing Fault-Tolerant Services Using the State Machine Approach: A Tutorial"](http://www.cs.cornell.edu/fbs/publications/smsurvey.pdf)**, *ACM Computing Surveys*, vol. 22, no. 4, pp. 299–319, December 1990.
[61]. Alexander Thomson, Thaddeus Diamond, Shu-Chun Weng, et al.: **["Calvin: Fast Distributed Transactions for Partitioned Database Systems"](http://cs.yale.edu/homes/thomson/publications/calvin-sigmod12.pdf)**, at *ACM International Conference on Management of Data (SIGMOD)*, May 2012.
[62]. Mahesh Balakrishnan, Dahlia Malkhi, Ted Wobber, et al.: **["Tango: Distributed Data Structures over a Shared Log"](http://research.microsoft.com/pubs/199947/Tango.pdf)**, at *24th ACM Symposium on Operating Systems Principles (SOSP)*, November 2013. **[doi:10.1145/2517349.2522732](http://dx.doi.org/10.1145/2517349.2522732)** 
[63]. Robbert van Renesse and Fred B. Schneider: **["Chain Replication for Supporting High Throughput and Availability"](http://static.usenix.org/legacy/events/osdi04/tech/full_papers/renesse/renesse.pdf)**, at *6th USENIX Symposium on Operating System Design and Implementation (OSDI)*, December 2004.
[64]. Leslie Lamport: **["How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs"](http://research-srv.microsoft.com/en-us/um/people/lamport/pubs/multi.pdf)**, *IEEE Transactions on Computers*, vol. 28, no. 9, pp. 690–691, September 1979. **[doi:10.1109/TC.1979.1675439](http://dx.doi.org/10.1109/TC.1979.1675439)** 
[65]. Enis Söztutar, Devaraj Das, and Carter Shanklin: **["Apache HBase High Availability at the Next Level"](http://hortonworks.com/blog/apache-hbase-high-availability-next-level/)**, *hortonworks.com*, January 22, 2015.
[66]. Brian F Cooper, Raghu Ramakrishnan, Utkarsh Srivastava, et al.: **["PNUTS: Yahoo!'s Hosted Data Serving Platform"](http://www.mpi-sws.org/%7Edruschel/courses/ds/papers/cooper-pnuts.pdf)**, at *34th International Conference on Very Large Data Bases (VLDB)*, August 2008. **[doi:10.14778/1454159.1454167](http://dx.doi.org/10.14778/1454159.1454167)** 
[67]. Tushar Deepak Chandra and Sam Toueg: **["Unreliable Failure Detectors for Reliable Distributed Systems"](http://courses.csail.mit.edu/6.852/08/papers/CT96-JACM.pdf)**, *Journal of the ACM*, vol. 43, no. 2, pp. 225–267, March 1996. **[doi:10.1145/226643.226647](http://dx.doi.org/10.1145/226643.226647)** 
[68]. Michael J. Fischer, Nancy Lynch, and Michael S. Paterson: **["Impossibility of Distributed Consensus with One Faulty Process"](https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf)**, *Journal of the ACM*, vol. 32, no. 2, pp. 374–382, April 1985. **[doi:10.1145/3149.214121](http://dx.doi.org/10.1145/3149.214121)** 
[69]. Michael Ben-Or: **"Another Advantage of Free Choice: Completely Asynchronous Agreement Protocols"**, at *2nd ACM Symposium on Principles of Distributed Computing (PODC)*, August 1983. **[doi:10.1145/800221.806707](http://dl.acm.org/citation.cfm?id=806707)** 
[70]. Jim N. Gray and Leslie Lamport: **"Consensus on Transaction Commit"**, *ACM Transactions on Database Systems (TODS)*, vol. 31, no. 1, pp. 133–160,March 2006. **[doi:10.1145/1132863.1132867](http://dx.doi.org/10.1145/1132863.1132867)** 
[71]. Rachid Guerraoui: **["Revisiting the Relationship Between Non-Blocking Atomic Commitment and Consensus"](https://pdfs.semanticscholar.org/5d06/489503b6f791aa56d2d7942359c2592e44b0.pdf)**, at *9th International Workshop on Distributed Algorithms (WDAG)*, September 1995. **[doi:10.1007/BFb0022140](http://dx.doi.org/10.1007/BFb0022140)** 
[72]. Thanumalayan Sankaranarayana Pillai, Vijay Chidambaram, Ramnatthan Alagappan, et al.: **["All File Systems Are Not Created Equal: On the Complexity of Crafting Crash-Consistent Applications"](http://research.cs.wisc.edu/wind/Publications/alice-osdi14.pdf)**, at *11th USENIX Symposium on Operating Systems Design and Implementation (OSDI)*, October 2014.
[73]. Jim Gray: **["The Transaction Concept: Virtues and Limitations"](https://jimgray.azurewebsites.net/papers/theTransactionConcept.pdf)**, at *7th International Conference on Very Large Data Bases (VLDB)*, September 1981.
[74]. Hector Garcia-Molina and Kenneth Salem: *"Sagas"*, at *ACM International Conference on Management of Data (SIGMOD)*, May 1987. **[doi:10.1145/38713.38742](http://dx.doi.org/10.1145/38713.38742)** 
[75]. C. Mohan, Bruce G. Lindsay, and Ron Obermarck: **["Transaction Management in the R&#42; Distributed Database Management System"](https://cs.brown.edu/courses/csci2270/archives/2012/papers/dtxn/p378-mohan.pdf)**, *ACM Transactions on Database Systems*, vol. 11, no. 4, pp. 378–396, December 1986. **[doi:10.1145/7239.7266](http://dx.doi.org/10.1145/7239.7266)** 
[76]. **["Distributed Transaction Processing: The XA Specification"](pubs.opengroup.org/onlinepubs/009680699/toc.pdf)**, X/Open Company Ltd., Technical Standard XO/CAE/91/300, December 1991. ISBN: 978-1-872-63024-3
[77]. Mike Spille: **["XA Exposed, Part II"](http://www.jroller.com/pyrasun/entry/xa_exposed_part_ii_schwartz)**, *jroller.com*, April 3, 2004.
[78]. Ivan Silva Neto and Francisco Reverbel: **["Lessons Learned from Implementing WS-Coordination and WS-AtomicTransaction"](http://www.ime.usp.br/%7Ereverbel/papers/icis2008.pdf)**, at *7th IEEE/ACIS International Conference on Computer and Information Science (ICIS)*, May 2008. **[doi:10.1109/ICIS.2008.75](http://dx.doi.org/10.1109/ICIS.2008.75)** 
[79]. James E. Johnson, David E. Langworthy, Leslie Lamport, and Friedrich H. Vogt: **"Formal Specification of a Web Services Protocol"**, at *1st International Workshop on Web Services and Formal Methods (WS-FM)*, February 2004. **[doi:10.1016/j.entcs.2004.02.022](http://dx.doi.org/10.1016/j.entcs.2004.02.022)** 
[80]. Dale Skeen: **["Nonblocking Commit Protocols"](http://www.cs.utexas.edu/%7Elorenzo/corsi/cs380d/papers/Ske81.pdf)**, at *ACM International Conference on Management of Data (SIGMOD)*, April 1981. **[doi:10.1145/582318.582339](http://dx.doi.org/10.1145/582318.582339)** 
[81]. Gregor Hohpe: **["Your Coffee Shop Doesn't Use Two-Phase Commit"](http://www.martinfowler.com/ieeeSoftware/coffeeShop.pdf)**, *IEEE Software*, vol. 22, no. 2, pp. 64–66, March 2005. **[doi:10.1109/MS.2005.52](http://dx.doi.org/10.1109/MS.2005.52)** 
[82]. Pat Helland: **["Life Beyond Distributed Transactions: An Apostate's Opinion"](http://adrianmarriott.net/logosroot/papers/LifeBeyondTxns.pdf)**, at *3rd Biennial Conference on Innovative Data Systems Research (CIDR)*, January 2007.
[83]. Jonathan Oliver: **["My Beef with MSDTC and Two-Phase Commits"](http://blog.jonathanoliver.com/my-beef-with-msdtc-and-two-phase-commits/)**, *blog.jonathanoliver.com*, April 4, 2011.
[84]. Oren Eini (Ahende Rahien): **["The Fallacy of Distributed Transactions"](http://ayende.com/blog/167362/the-fallacy-of-distributed-transactions)**, *ayende.com*, July 17, 2014.
[85]. Clemens Vasters: **["Transactions in Windows Azure (with Service Bus) – An Email Discussion"](https://blogs.msdn.microsoft.com/clemensv/2012/07/30/transactions-in-windows-azure-with-service-bus-an-email-discussion/)**, *vasters.com*, July 30, 2012.
[86]. **["Understanding Transactionality in Azure"](https://docs.particular.net/nservicebus/azure/understanding-transactionality-in-azure)**, NServiceBus Documentation, Particular Software, 2015.
[87]. Randy Wigginton, Ryan Lowe, Marcos Albe, and Fernando Ipar: **["Distributed Transactions in MySQL"](https://www.percona.com/live/mysql-conference-2013/sites/default/files/slides/XA_final.pdf)**, at *MySQL Conference and Expo*, April 2013.
[88]. Mike Spille: **["XA Exposed, Part I"](http://www.jroller.com/pyrasun/entry/xa_exposed)**, *jroller.com*, April 3, 2004.
[89]. Ajmer Dhariwal: **["Orphaned MSDTC Transactions (-2 spids)"](http://www.eraofdata.com/orphaned-msdtc-transactions-2-spids/)**, *eraofdata.com*, December 12, 2008.
[90]. Paul Randal: **["Real World Story of DBCC PAGE Saving the Day"](http://www.sqlskills.com/blogs/paul/real-world-story-of-dbcc-page-saving-the-day/)**, *sqlskills.com*, June 19, 2013.
[91]. **["in-doubt xact resolution Server Configuration Option"](https://technet.microsoft.com/en-us/library/ms179586(v=sql.110).aspx)**, SQL Server 2016 documentation, Microsoft, Inc., 2016.
[92]. Cynthia Dwork, Nancy Lynch, and Larry Stockmeyer: **["Consensus in the Presence of Partial Synchrony"](http://www.net.t-labs.tu-berlin.de/%7Epetr/ADC-07/papers/DLS88.pdf)**, *Journal of the ACM*, vol. 35, no. 2, pp. 288– 323, April 1988. **[doi:10.1145/42282.42283](http://dx.doi.org/10.1145/42282.42283)** 
[93]. Miguel Castro and Barbara H. Liskov: **["Practical Byzantine Fault Tolerance and Proactive Recovery"](http://zoo.cs.yale.edu/classes/cs426/2012/bib/castro02practical.pdf)**, *ACM Transactions on Computer Systems*, vol. 20, no. 4, pp. 396–461, November 2002. **[doi:10.1145/571637.571640](http://dx.doi.org/10.1145/571637.571640)** 
[94]. Brian M. Oki and Barbara H. Liskov: **["Viewstamped Replication: A New Primary Copy Method to Support Highly-Available Distributed Systems"](http://www.cs.princeton.edu/courses/archive/fall11/cos518/papers/viewstamped.pdf)**, at *7th ACM Symposium on Principles of Distributed Computing (PODC)*, August 1988. **[doi:10.1145/62546.62549](http://dx.doi.org/10.1145/62546.62549)** 
[95]. Barbara H. Liskov and James Cowling: **["Viewstamped Replication Revisited"](http://pmg.csail.mit.edu/papers/vr-revisited.pdf)**, Massachusetts Institute of Technology, Tech Report MIT-CSAIL-TR-2012-021, July 2012.
[96]. Leslie Lamport: **["The Part-Time Parliament"](http://research.microsoft.com/en-us/um/people/lamport/pubs/lamport-paxos.pdf)**, *ACM Transactions on Computer Systems*, vol. 16, no. 2, pp. 133–169, May 1998. **[doi:10.1145/279227.279229](http://dx.doi.org/10.1145/279227.279229)** 
[97]. Leslie Lamport: **["Paxos Made Simple"](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)**, *ACM SIGACT News*, vol. 32, no. 4, pp. 51–58, December 2001.
[98]. Tushar Deepak Chandra, Robert Griesemer, and Joshua Redstone: **["Paxos Made Live – An Engineering Perspective"](http://www.read.seas.harvard.edu/%7Ekohler/class/08w-dsi/chandra07paxos.pdf)**, at *26th ACM Symposium on Principles of Distributed Computing (PODC)*, June 2007.
[99]. Robbert van Renesse: **["Paxos Made Moderately Complex"](www.cs.cornell.edu/courses/cs7412/2011sp/paxos.pdf)**, *cs.cornell.edu*, March 2011.
[100]. Diego Ongaro: **["Consensus: Bridging Theory and Practice"](https://ramcloud.stanford.edu/~ongaro/thesis.pdf)**, PhD Thesis, Stanford University, August 2014.
[101]. Heidi Howard, Malte Schwarzkopf, Anil Madhavapeddy, and Jon Crowcroft: **["Raft Refloated: Do We Have Consensus?"](http://www.cl.cam.ac.uk/%7Ems705/pub/papers/2015-osr-raft.pdf)**, *ACM SIGOPS Operating Systems Review*, vol. 49, no. 1, pp. 12–21, January 2015. **[doi:10.1145/2723872.2723876](http://dx.doi.org/10.1145/2723872.2723876)** 
[102]. André Medeiros: **["ZooKeeper's Atomic Broadcast Protocol: Theory and Practice"](http://www.tcs.hut.fi/Studies/T-79.5001/reports/2012-deSouzaMedeiros.pdf)**, Aalto University School of Science, March 20, 2012.
[103]. Robbert van Renesse, Nicolas Schiper, and Fred B. Schneider: **["Vive La Différence: Paxos vs. Viewstamped Replication vs. Zab"](http://arxiv.org/abs/1309.5671)**, *IEEE Transactions on Dependable and Secure Computing*, vol. 12, no. 4, pp. 472–484, September 2014. **[doi:10.1109/TDSC.2014.2355848](http://dx.doi.org/10.1109/TDSC.2014.2355848)** 
[104]. Will Portnoy: **["Lessons Learned from Implementing Paxos"](http://blog.willportnoy.com/2012/06/lessons-learned-from-paxos.html)**, *blog.willportnoy.com*, June 14, 2012.
[105]. Heidi Howard, Dahlia Malkhi, and Alexander Spiegelman: **["Flexible Paxos: Quorum Intersection Revisited"](https://arxiv.org/abs/1608.06696.pdf)**, *arXiv:1608.06696*, August 24, 2016.
[106]. Heidi Howard and Jon Crowcroft: **["Coracle: Evaluating Consensus at the Internet Edge"](http://www.sigcomm.org/sites/default/files/ccr/papers/2015/August/2829988-2790010.pdf)**, at *Annual Conference of the ACM Special Interest Group on Data Communication (SIGCOMM)*, August 2015. **[doi:10.1145/2829988.2790010](http://dx.doi.org/10.1145/2829988.2790010)** 
[107]. Kyle Kingsbury: **["Call Me Maybe: Elasticsearch 1.5.0"](https://aphyr.com/posts/323-call-me-maybe-elasticsearch-1-5-0)**, *aphyr.com*, April 27, 2015.
[108]. Ivan Kelly: **["BookKeeper Tutorial"](https://github.com/apache/bookkeeper)**, *github.com*, October 2014.
[109]. Camille Fournier: **["Consensus Systems for the Skeptical Architect"](http://www.ustream.tv/recorded/61483409)**, at *Craft Conference*, Budapest, Hungary, April 2015.
[110]. Kenneth P. Birman: **["A History of the Virtual Synchrony Replication Model"](https://www.truststc.org/pubs/713.html)**, in *Replication: Theory and Practice*, Springer LNCS vol. 5959, ch. 6, pp. 91–120, 2010. ISBN: 978-3-642-11293-5, **[doi:10.1007/978-3-642-11294-2_6](http://dx.doi.org/10.1007/978-3-642-11294-2_6)** 

