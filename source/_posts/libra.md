---
title: 深入理解 Libra
date: 2020-05-20 18:12:19
author: Rootkit
top: false
mathjax: false
categories: Blockchain
tags:
  - Blockchain
  - Libra
---



### 1. 共识

区块链技术中，共识算法是其中核心的一个组成部分。它包含两层含义：

1. 多个节点对某个数据达成一致共识。
2. 多个节点对多个数据的顺序达成一致共识。

共识算法主要用在以下三种场景：

+ 私链
  + 私链的共识算法即传统分布式系统里的共识算法，比如 zookeeper 的 zab 协议，就是类 paxos 算法的一种。私链一般不考虑集群中存在作恶节点，只考虑因为系统或者网络原因导致的故障节点。
+ 联盟链
  + 联盟链代表项目是 Fabric，它使用的就是 PBFT(实用拜占庭) 算法。联盟链除了需要考虑集群中存在故障节点，还需要考虑集群中存在作恶节点。对于联盟链，每个新加入的节点都是需要验证和审核。
+ 公链
  + 公链不仅需要考虑网络中存在故障节点，还需要考虑作恶节点，这一点和联盟链是类似的。和联盟链最大的区别就是，公链中的节点可以很自由的加入或者退出，不需要严格的验证和审核。

### 2. 通讯模型

分布式系统首先要明确其前置的`通讯模型`（也称为`时间模型`）的约定。比较常见的有**同步网络模型**、**异步网络模型**和**部分同步网络模型**。

#### **2.1 同步网络模型**

节点所发出的消息，在一个确定的时间内，肯定会到达目标节点。

在同步网络模型中：

1. 进程间的消息通讯、传输延时有界。认为通讯耗时是有确定范围的；
2. 每个进程的处理速度是确定的。我们可以确切知道进程中每步算法的耗时。

我们可以推导出：

- 每个进程间的时钟是同步的，因为前面的定义**1**、**2**，所以我们可以使用通讯来同步各个机器上的时钟，使得各个机器的时钟误差在 ΔT 内。
- 如果一个请求的应答超过 RTT+ ΔT 请求处理耗时，则可以判定对端异常。

`同步网络模型`使用起来最简单，是理想的网络模型，在实际环境中并不存在。

#### **2.2 异步网络模型**

节点所发出的消息，不能确定一定会到达目标节点。

在`异步网络模型`中，我们引用 **FLP 不可能性**[^6]中的定义：

1. 进程间的消息通讯、传输延时没有上界；
2. 每个进程的处理速度是不确定的；
3. 每个机器上没有同步时钟（即不存在原子钟这种东西），时间流逝速度可能也不同。

因此我们可以推导出：

- 各个机器上的时钟没有可参考性，因为根据上面 3 每个机器不能自发保持时间的一致性，并且因为1、2，机器时间也无法同步时钟在一个有界的误差之内。所以依赖超时机制的算法并不可用。
- 因为1、2，当一个请求在本地时钟上超时后，我们无法判断这个请求是否是因为对端异常造成的。这是一个经典的**两军问题**[^1]场景，故，我们无法对其他实例进行故障探测。

从定义中 3，我们引出两个概念，但是这两个概念原本是集成电路[^7][^8]中的概念，但是我们在分布式系统中重新扩展一下：

- `clock drift(时钟漂移)`：相关节点上的时钟以不同的速率运行。在 P1 节点上经过 ΔT1 的同时，P2 节点经过了ΔT2，但是 ΔT1≠ΔT2。
- `clock skew(时钟偏移)`：相关节点都引用了同一个时间源（比如通过 NTP 服务），但是由于这个时间源将授时信号同步不同节点时的耗时不同（比如网络传输耗时），造成同一时刻不同节点间产生了时间差。这是一个比`clock drift(时钟漂移)`更加严格的要求，因为如果一个系统内部的 TSkew 存在上界TShewMax ，那一定能通过不断的校时、保持时钟同步，使得 TDrift 保持在 TShewMax 以内。

异步网络模型是一个最理想的**最差**网络模型，但是其复杂度又远超我们实际情形。

#### **2.3 部分同步网络模型**

节点发出的消息，虽然会有延迟，但是最终会到达目标节点。

我们现实中遇见的网络模型通常介于同步网络模型和异步网络模型两者之间。因为在我们所知的大部分系统，在大部分时间内：

- 进程间的消息通讯、传输延时是有上界的；只有在网络过载、网络分区故障时，才没有上界；
- 每个进程的处理速度是确定的；只有在发生 GC 、磁盘 IO 阻塞等异常情况时，每个进程的处理速度才不可确定。
- 每个机器上的时钟我们可以认为是基本同步的，比如我们可以使用 NTP[^9]来同步机器时间，而且多数机器上有独立的时钟芯片[^10]，我们也可以粗略认为各个机器上时间流逝的速度是相同的。但是严格要求时序的系统除外。

### 3. Raft 算法

raft 中节点有三种状态：

* Follower （跟随者）  被动的只是响应来自领导者和候选人的请求 
* Candidate （候选人）被选出一个新的Leader 
* Leader（领导者）  处理所有来自客户端的请求

集群中的一个节点在某一时刻只能是这三种状态的其中一种，这三种状态是可以随着时间和条件的变化而互相转换的。

raft 算法主要有两个过程：

+ 领导者选举
+ 日志复制
  + 记录日志
  + 提交数据 

raft 算法支持最大的容错故障节点 (n-1)/2，其中 n 为集群中总的节点数量。

> raft 算法只支持容错故障节点，假设集群总节点数为 n，故障节点为 f，根据小数服从多数的原则，集群里正常节点只需要比 f 个节点再多一个节点，即 f+1个节点，正确节点的数量就会比故障节点数量多，那么集群就能达成共识。因此 raft 算法支持的最大容错节点数量是 (n-1)/2。

可参考[动画](http://thesecretlivesofdata.com/raft/)理解

#### 3.1 算法复杂度

Raft 算法核心是日志复制这个过程，这个过程分两个阶段：一个是日志记录，一个是提交数据。两个过程都只需要领导者发送消息给跟随者节点，跟随者节点返回消息给领导者节点即可完成，跟随者节点之间是无需沟通的。所以如果集群总节点数为 n，对于日志记录阶段，通信次数为 n-1，对于提交数据阶段，通信次数也为 n-1，总通信次数为 2n-2，因此 raft 算法复杂度为O(n)。

### 4. 两军问题

两军问题[^1]，又称为“两军悖论”，是计算机通信领域的一个思想实验，主要用来描述在一个不可靠的通信链上试图通过通信达成一致是存在缺陷与困难的，适用于任何可能通信失败情况下的两点通信，两军问题被证明无解。如果通信信道是可靠的，我们只需三次握手就可以解决问题，你将消息发给战友，战友确认回复，你再确认收到回复，经过这三次握手，基本就可以发起进攻，但是两军问题面临的通信通道是不可靠的，无论多少次握手也无法保证最后一次通信准确送达，最后一次通信的发送方就会一直面临着冒着失败风险的行动。

### 5. 拜占庭将军问题

拜占庭将军问题是一个共识问题: 首先由 Leslie Lamport 与另外两人在1982年提出，被称为 The Byzantine Generals Problem[^2]或者 Byzantine Failure。核心描述是军中可能有叛徒，却要保证进攻一致，由此引申到计算领域，发展成了一种容错理论，即拜占庭容错（BFT，Byzantine Fault Tolerance）。简单来说，拜占庭容错（BFT）是能够抵抗拜占庭将军问题导致的一系列失败的系统属性。 这意味着即使某些节点出现故障或恶意行为，拜占庭容错系统也能够继续运行。

拜占庭将军的问题有多种可能的解决方案，因此，有多种方法可以构建拜占庭容错系统。同样地，区块链有各种不同的方法来实现拜占庭容错，这就是我们说的共识算法。

#### 5.1 问题描述

拜占庭帝国想要进攻一个强大的敌人，为此派出了10支军队去包围这个敌人。这个敌人虽不比拜占庭帝国，但也足以抵御5支常规拜占庭军队的同时袭击。基于一些原因，这10支军队不能集合在一起单点突破，必须在分开的包围状态下同时攻击。他们任一支军队单独进攻都毫无胜算，除非有至少6支军队同时袭击才能攻下敌国。他们分散在敌国的四周，依靠通信兵相互通信来协商进攻意向及进攻时间。困扰这些将军的问题是，他们不确定他们中是否有叛徒，叛徒可能擅自变更进攻意向或者进攻时间。在这种状态下，拜占庭将军们能否找到一种分布式的协议来让他们能够远程协商，从而赢取战斗？这就是著名的拜占庭将军问题。

需要明确，拜占庭将军问题中并不去考虑通信兵是否会被截获或无法传达信息等问题，即消息传递的信道绝无问。

Lamport已经证明了**在消息可能丢失的不可靠信道上试图通过消息传递的方式达到一致性是不可能的**。所以，在研究拜占庭将军问题的时候，我们已经假定了信道是没有问题的，并在这个前提下，去做一致性和容错性相关研究。如果需要考虑信道是有问题的，这涉及到了另一个相关问题：**两军问题**。

拜占庭失效指一方向另一方发送消息，另一方没有收到，或者收到了错误的信息的情形。在容错的分布式计算中，拜占庭失效可以是分布式系统中算法执行过程中的任意一个错误。

Lamport提出并论证了口头算法和书面算法两种解决方法。

#### 5.2 口头协议

首先，我们明确什么是口头协议。我们将满足以下三个条件的方式称为口头协议：

```
 (1) 每个被发送的消息都能够被正确的投递

 (2) 信息接收者知道是谁发送的消息

 (3) 能够知道缺少的消息
```

简而言之，信道绝对可信，且消息来源可知。但要注意的是，口头协议并不会告知消息的上一个来源是谁。

Lamport 论证得出结论：采用口头协议，若叛徒数少于 1/3，则拜占庭将军问题可解。也就是说，若叛徒数为m，当将军总数n至少为 3m+1 时，问题可解。

#### 5.3 书面协议

揭示了口头协议的缺点是消息不能追本溯源，这使得口头协议必须在四模冗余的情况下才能保证正确。由此引入了书面协议。

在口头协议的三个条件之上，再添加一个条件，使之成为书面协议。

```
 （a）签名不可伪造，一旦被篡改即可发现，而叛徒的签名可被其他叛徒伪造；
 （b）任何人都可以验证签名的可靠性。  
```

可以论证：对于任意m，最多只有m个背叛者情况下，算法SM(m)能解决拜占庭将军问题。

书面协议的本质就是引入了签名系统，这使得所有消息都可追本溯源。这一优势，大大节省了成本，他化解了口头协议中1/3要求，只要采用了书面协议，忠诚的将军就可以达到一致。

### 6. 实用拜占庭容错（PBFT）

实用拜占庭容错协议[^3]（PBFT，Practical Byzantine Fault Tolerance）是Miguel Castro (卡斯特罗)和Barbara Liskov（利斯科夫）在1999年提出来的，解决了原始拜占庭容错算法效率不高的问题，将算法复杂度由指数级降低到多项式级，使得拜占庭容错算法在实际系统应用中变得可行。

PBFT 是一个具有二轮投票的三阶段协议，每个视图(View)都会有一个特定的节点作为领导节点(Primary/Leader)，负责通知所有节点进入投票流程。各节点则会经历 Pre-prepare/Prepare/Commit 这三个阶段，并依据接收的讯息决定是否投票/进入下一阶段，每个节点投完票后将讯息发给所有其他的节点。

若个节点在两阶段投票之后取得多数共识，则各节点可以更新本机的状态，结束这一回合。视图变换(view change)仅当多数节点发起时执行，当目前的领导节点并未正常执行任务时，这可以替换当前的领导节点，保证协议正常运作。

PBFT 是一种状态机副本复制算法，即服务作为状态机进行建模，状态机在分布式系统的不同节点进行副本复制。每个状态机的副本都保存了服务的状态，同时也实现了服务的操作。将所有的副本组成的集合使用大写字母 R 表示，使用 0 到 R 减1的整数表示每一个副本。为了描述方便，假设 R=3f+1，这里 f 是有可能失效的副本的最大个数。尽管可以存在多于 3f+1 个副本，但是额外的副本除了降低性能之外不能提高可靠性。

主节点由公式`p = v mod R`计算得到，这里 v 是视图编号，p 是副本编号，R 是副本集合的个数。当主节点失效的时候就需要启动视图更换（view change）过程。Viewstamped Replication 算法和 Paxos 算法就是使用类似方法解决良性容错的。

每个副本节点的状态都包含了服务的整体状态，副本节点上的**消息日志(message log)**包含了该副本节点**接受(accepted)**的消息，并且使用一个整数表示副本节点的当前视图编号。

> 为什么节点数需要大于 3f+1 
>
> 因为最坏的情况是：f 个节点是有问题的，由于到达顺序的问题，有可能 f 个有问题的节点比正常的 f 个节点先返回消息，又要保证收到的正常的节点比有问题的节点多，所以需要满足n-f-f>f => n>3f，所以至少3f+1个节点。
>
> <img src ="/images/2019/pbft_1.png"/>

#### 6.1 系统模型

一组节点构成状态机复制系统，一个节点作为主节点（privary），其他节点作为备份节点(back-ups)。某个节点作为主节点时，这称为系统的一个 view。当节点出了问题，就进行 view 更新，切换到下一个节点担任主节点。主节点更替不需要选举过程，而是采用 round-robin 方式。

在系统的主节点接收 client 发来的请求，并产生 pre-prepare 消息，进入共识流程。

我们需要系统满足如下两个条件：

+ 在一个给定状态上的操作， 产生一样的执行结果
+ 每个节点都有一样的起始状态

要保证 non-fault 节点对于执行请求的**全局顺序**达成一致。

#### 6.2 安全性(safety) / 活性(liveness)

- **safety**: 坏的事情不会发生，即共识系统不能产生错误的结果，比如一部分节点说yes，另一部分说no。在区块链的语义下，指的是不会分叉。
- **liveness**: 好的事情一定会发生，即系统一直有回应，在区块链的语义下，指的是共识会持续进行，不会卡住。假如一个区块链系统的共识卡在了某个高度，那么新的交易是没有回应的，也就是不满 liveness。

#### 6.3 PBFT算法的步骤

1. 取一个节点作为主节点，其他的节点作为备份；
2. 客户端向主节点发送使用服务操作的请求；
3. 主节点通过广播将请求发送给其他节点；
4. 所有节点执行请求并将结果发回客户端；
5. 客户端需要等待 f+1 个不同节点发回相同的结果，作为整个操作的最终结果。

同所有的状态机副本复制技术一样，PBFT 对每个节点提出了两个限定条件：

1. 所有节点必须是确定性的。也就是说，在给定状态和参数相同的情况下，操作执行的结果必须相同；
2. 所有节点必须从相同的状态开始执行。

这两个限定条件下，即使失效的节点存在，PBFT 算法对所有非失效节点的请求执行总顺序达成一致，从而保证安全性。

```假设节点总数为3f+1，f为拜赞庭错误节点：
1. 当节点发现 leader 作恶时，通过算法选举其他的 replica 为 leader。

2. leader 通过 pre-prepare 消息把它选择的 value 广播给其他 replica 节点，其他的 replica 节点如果接受则发送 prepare，如果失败则不发送。

3. 一旦2f个节点接受 prepare 消息，则节点发送 commit 消息。

4. 当2f+1个节点接受 commit 消息后，代表该 value 值被确定。
```

如下图表示了4个节点，0为leader，同时节点3为fault节点，该节点不响应和发出任何消息，C为客户端。最终节点状态达到commited时，表示该轮共识成功达成。



<img src ="/images/2019/pbft.png"/>



#### 6.4 客户端C

客户端c向主节点发送`<REQUEST,o,t,c>`请求执行状态机操作o，这里时间戳t用来保证客户端请求只会执行一次。客户端c发出请求的时间戳是全序排列的，后续发出的请求比早先发出的请求拥有更高的时间戳。例如，请求发起时的本地时钟值可以作为时间戳。

每个由副本节点发给客户端的消息都包含了当前的视图编号，使得客户端能够跟踪视图编号，从而进一步推算出当前主节点的编号。客户端通过点对点消息向它自己认为的主节点发送请求，然后主节点自动将该请求向所有备份节点进行广播。

副本发给客户端的响应为`<REPLY,v,t,c,i,r>`，v是视图编号，t是时间戳，i是副本的编号，r是请求执行的结果。

客户端等待f+1个从不同副本得到的同样响应，同样响应需要保证签名正确，并且具有同样的时间戳t和执行结果r。这样客户端才能把r作为正确的执行结果，因为失效的副本节点不超过f个，所以f+1个副本的一致响应必定能够保证结果是正确有效的。

如果客户端没有在有限时间内收到回复，请求将向所有副本节点进行广播。如果请求已经在副本节点处理过了，副本就向客户端重发一遍执行结果。如果请求没有在副本节点处理过，该副本节点将把请求转发给主节点。如果主节点没有将该请求进行广播，那么就有认为主节点失效，如果有足够多的副本节点认为主节点失效，则会触发一次视图变更。

本文假设客户端会等待上一个请求完成才会发起下一个请求，但是只要能够保证请求顺序，可以允许请求是异步的。

#### 6.5 预准备(pre-prepare)

在预准备阶段，主节点分配一个序列号 n 给收到的请求，然后向所有节点群发预准备消息，预准备消息的格式为`<<PRE-PREPARE,v,n,d>,m>`，这里 v 是视图编号，m 是客户端发送的请求消息，d 是请求消息 m 的摘要。

请求本身是不包含在预准备的消息里面的，这样就能使预准备消息足够小，因为**预准备消息的目的是作为一种证明，确定该请求是在视图 v 中被赋予了序号n，从而在视图变更的过程中可以追索**。另外一个层面，将“请求排序协议”和“请求传输协议”进行解耦，有利于对消息传输的效率进行深度优化。

只有满足以下条件，各个备份节点才会接受一个预准备消息：

1. 请求和预准备消息的签名正确，并且 d 与 m 的摘要一致。
2. 当前视图编号是 v。
3. 该备份节点从未在视图 v 中接受过序号为 n 但是摘要 d 不同的消息 m。
4. 预准备消息的序号 n 必须在水线（watermark）上下限 h 和 H 之间。

水线存在的意义在于防止一个失效节点使用一个很大的序号消耗序号空间。

#### 6.6 准备(prepare)

如果备份节点i接受了预准备消息`<<PRE-PREPARE,v,n,d>,m>`，则进入准备阶段。在准备阶段的同时，该节点向所有副本节点发送准备消息`<PREPARE,v,n,d,i>`，并且将预准备消息和准备消息写入自己的消息日志。如果看预准备消息不顺眼，就什么都不做。

包括主节点在内的所有副本节点在收到准备消息之后，对消息的签名是否正确，视图编号是否一致，以及消息序号是否满足水线限制这三个条件进行验证，如果验证通过则把这个准备消息写入消息日志中。

我们定义准备阶段完成的标志为副本节点i将`(m,v,n,i)`记入其消息日志，其中 m 是请求内容，预准备消息 m在视图 v 中的编号 n，以及 2f 个从不同副本节点收到的与预准备消息一致的准备消息。每个副本节点验证预准备和准备消息的一致性主要检查：视图编号 v 、消息序号 n 和摘要 d。

预准备阶段和准备阶段确保所有正常节点对同一个视图中的请求序号达成一致。接下去是对这个结论的形式化证明：如果 `prepared(m,v,n,i)` 为真，则`prepared(m',v,n,j)`必不成立，这就意味着至少 f+1 个正常节点在视图 v 的预准备或者准备阶段发送了序号为 n 的消息 m。

#### 6.7 确认(commit)

当 (m,v,n,i) 条件为真的时候，副本 i 将`<COMMIT,v,n,D(m),i>` 向其他副本节点广播，于是就进入了确认阶段。

每个副本接受确认消息的条件是：

1. 签名正确；
2. 消息的视图编号与节点的当前视图编号一致；
3. 消息的序号 n 满足水线条件，在 h 和 H 之间。一旦确认消息的接受条件满足了，则该副本节点将确认消息写入消息日志中。（补充：需要将针对某个请求的所有接受的消息写入日志，这个日志可以是在内存中的）。

确认阶段保证了以下这个不变式（invariant）：对某个正常节点 i 来说，如果 committed-local(m,v,n,i) 为真则 committed(m,v,n) 也为真。这个不变式和视图变更协议保证了所有正常节点对本地确认的请求的序号达成一致，即使这些请求在每个节点的确认处于不同的视图。更进一步地讲，这个不变式保证了任何正常节点的本地确认最终会确认 f+1 个更多的正常副本。

每个副本节点 i 在 committed-local(m,v,n,i) 为真之后执行 m 的请求，并且i的状态反映了所有编号小于 n 的请求依次顺序执行。这就确保了所有正常节点以同样的顺序执行所有请求，这样就保证了算法的正确性（safety）。在完成请求的操作之后，每个副本节点都向客户端发送回复。副本节点会把时间戳比已回复时间戳更小的请求丢弃，以保证请求只会被执行一次。

#### 6.8 检查点/垃圾回收

为了节省内存，系统需要一种将日志中的无异议消息记录删除的机制。为了保证系统的安全性，节点在删除自己的消息日志前，需要确保至少 f+1 个正常节点执行了消息对应的请求，并且可以在视图变更时向其他节点证明。另外，如果一些节点错过部分消息，但是这些消息已经被所有正常节点删除了，这就需要通过传输部分或者全部服务状态实现该节点的同步。因此，节点同样需要证明状态的正确性。

在每一个操作执行后都生成这样的证明是非常消耗资源的。因此，证明过程只有在请求序号可以被某个常数（比如100）整除的时候才会周期性地进行。我们将这些请求执行后得到的状态称作检查点（checkpoint），并且将具有证明的检查点称作稳定检查点（stable checkpoint）。

节点保存了服务状态的多个逻辑拷贝，包括最新的稳定检查点，零个或者多个非稳定的检查点，以及一个当前状态。写时复制技术可以被用来减少存储额外状态拷贝的空间开销。

检查点的正确性证明的生成过程如下：当节点i生成一个检查点后，向其他节点广播检查点消息`<CHECKPOINT,n,d,i>`，这里 n 是最近一个影响状态的请求序号，d是状态的摘要。每个节点都默默地在各自的日志中收集并记录其他节点发过来的检查点消息，直到收到来自 2f+1 个不同节点的具有相同序号n和摘要d的检查点消息。这2f+1 个消息就是这个检查点的正确性证明。

具有证明的检查点成为稳定检查点，然后从节点就可以将所有序号小于等于n的预准备、准备和确认消息从日志中删除。同时也可以将之前的检查点和检查点消息一并删除。

检查点协议可以用来更新水线（watermark）的高低值（h和H），这两个高低值限定了可以被接受的消息。水线的低值h与最近稳定检查点的序列号相同，而水线的高值H=h+k，k 需要足够大才能使节点不至于为了等待稳定检查点而停顿。加入检查点每100个请求产生一次，k的取值可以是200。

#### 6.9 视图更换(view change)

当主节点挂了（超时无响应）或者从节点集体认为主节点是问题节点时，就会触发 view change 事件，view change完成后，视图编号将会加1。

下图展示 view change 的三个阶段流程：

<img src ="/images/2019/pbft_2.png"/>

如图所示，view change 会有三个阶段，分别是 view-change，view-change-ack 和 new-view 阶段。从节点认为主节点有问题时，会向其它节点发送 view-change 消息，当前存活的节点编号最小的节点将成为新的主节点。当新的主节点收到2f个其它节点的 view-change 消息，则证明有足够多人的节点认为主节点有问题，于是就会向其它节点广播 new-view 消息。注意：从节点不会发起 new-view 事件。对于主节点，发送new-view 消息后会继续执行上个视图未处理完的请求，从 pre-prepare 阶段开始。其它节点验证 new-view 消息通过后，就会处理主节点发来的 pre-prepare 消息，这时执行的过程就是前面描述的 PBFT 过程。到这时，正式进入 v+1（视图编号加1）的视图了。



#### 6.10 算法复杂度

PBFT 算法核心流程有三个阶段，分别是 pre-prepare（预准备）阶段，prepare（准备）阶段和 commit（提交）阶段。对于 pre-prepare 阶段，主节点广播 pre-prepare 消息给其它节点即可，因此通信次数为 n-1；对于 prepare 阶段，每个节点如果同意请求后，都需要向其它节点再 广播 parepare消息，所以总的通信次数为$n*(n-1)$，即$n^2-n$；对于 commit 阶段，每个节点如果达到 prepared 状态后，都需要向其它节点广播 commit 消息，所以总的通信次数也为$n*(n-1)$，即$n^2-n$。所以总通信次数为$(n-1)+(n^2-n)+(n^2-n)$，即$2n^2-n-1$，因此 pbft 算法复杂度为$O(n^2)$。



### 7. HotStuff

一句话描述 HotStuff ： **HotStuff 是一个在部分同步网络模型下的基于主节点的拜占庭容错共识协议。** 

> We present HotStuff, a leader-based Byzantine fault-tolerant replication protocol for the partially synchronous model.



#### 7.1 PBFT 缺点

原始的 PBFT 算法在设计时考虑的是部署在本地的只有 n = 4 或者 n = 7 个节点的网络，所以 PBFT 算法在现有的网络环境下有很大的缺陷。PBFT 中每个节点都需要与其他的节点进行 P2P 的共识同步，因此随着节点数量的增多，整体系统的性能会发生线性的速度下降。PBFT 由于其封闭性（节点数目提前确定并互相联通）和高性能开销($O(n^2)$的消息复杂度)，复杂的 view change 算法和开销，当节点数目 n = 2000 时，每次共识消息量将会爆炸到 4,000,000 ，已经不太适合现有的需求。

<img src ="/images/2019/hotstuff_0.png"/>

####  7.2 HotStuff 流程

HotStuff **将视图切换流程和正常流程进行合并**，即不再有单独的视图切换流程，降低了视图切换的复杂度。在 HotStuff 中切换视图时，系统中的某个节点也无需再确认“足够多的节点希望进行视图切换”这一消息后再通知新的主节点，它直接切换到新视图并通知新的主节点。HotStuff 把确认“足够多的节点希望进行视图切换”这一消息的行为放进了正常流程中。这一做法比较新颖，但必然会给正常流程引入新的确认阶段。因此，HotStuff 把 PBFT 的两阶段确认扩展成了三阶段确认。

HotStuff 以 prepare 阶段作为协议的开始阶段。在这一阶段中，当主节点收集到足够的节点发来新视图请求后，它开始新视图并提出自己的状态迁移要求，发送 prepare 消息给其它节点。系统中的其它节点在接收到 prepare 消息后，验证其合法性并进行如下三阶段确认：

1. **pre-commit** 阶段：其它节点对 prepare 消息进行投票。在收到足够多的投票后，主节点向所有节点广播 pre-commit 消息，向它节点表明足够多的节点确认了此次状态迁移的要求。 
2. **commit** 阶段：其它节点对 pre-commit 消息进行投票。在收到足够多的投票后，主节点向所有节点广播 commit 消息。此时，收到 commit 消息的节点可以锁定当前状态迁移要求以便即使视图切换也可以顺利达成共识。
3. **decide** 阶段：其它节点对 commit 消息进行投票。在收到足够多的投票后，主节点向所有节点广播 decide 消息。当某个节点收到 decide 消息后将执行状态迁移，并开始新的视图。

<img src ="/images/2019/hotstuff_2.png"/>

#### 7.3 HotStuff 主要优化



##### 7.3.1 通信复杂度优化

HotStuff 每次通信都依靠主节点。节点不再通过 p2p 网络将消息广播给其它节点，而是将消息发送给主节点，由主节点处理后发送给其它节点, 系统的通信复杂度得到了大大降低。

传统 BFT 达成共识的方法是两轮共识，其中第一轮 prepare ，第二轮 commit。很多将 BFT 用于区块链的项目仍旧采取**先做两轮通信，然后达成共识，最后上链**的模式，而 Hotstuff 采用的是**先上链，在区块中加入门限签名(threshold signature)**，于是在 n 个区块之后就可以视为通过了 n 轮的通信达成共识。所以根本就不需要再去区分所谓 prepare，commit 这两轮通信的区别了，只需要简单地把每一轮节点的行为定义成**leader负责出块和收集签名**，然后**其他节点负责对leader出的块进行签名**，然后，只要收集到了2f+1 个签名，leader 就可以出一个块，然后后面有 n 个块就相当于达成了共识。这点的好处在于，O(n) 的通信复杂度可以让诚实节点知道 **我知道消息 m 将成为共识**，但是必须要 $O(n^2)$ 的通信才能让每个诚实节点都确信 **我还知道所有诚实节点也知道消息m是共识** ，而通过 leader 收集签名并出块这种方法，当所有人看到区块 b 的时候，诚实节点会知道 **我知道b是共识**，而在看到 b 后一块 b' 的时候，诚实节点等于知道了**所有签名的人也都知道了 b 是共识**。于是，每次出块的时候都只需要 O(n) 的消息复杂度，但是，在一个诚实 leader 和聚合签名的帮助下，通过两轮的O(n)消息复杂度，我们达到了之前 $O(n^2)$ 的效果。

##### 7.3.2 引入门限签名（threshold signature）

为了提升效率，一个直觉的思路是：**避免 $O(n^2)$ 的通讯**。我们可以指定网络中的某节点作为协调者来发送/接收每个节点的投票，这样每个节点都只需要向协调者发送讯息即可，从而避免了 $O(n^2)$ 的通讯。然而，在这样的情境下，协调者有作恶的可能，因为协调者可以在未确实接收到指定数量的讯息前便执行下一轮投票或者进行状态更新。

我们可以使用门限签名来保证协调者的正当行为，门限签名可以保证：需集合**超过门限数量(t-of-n)的签名才有效**。也就是说，我们可以指定：唯有当协调者集合 2f+1 个签名后，协调者才能带着合法的签名继续推进共识，这在之前已经有很多项目采用这种做法，比如：Dfinity。

<img src ="/images/2019/hotstuff_4.jpg"/>

##### 7.3.3 流水线作业

其它节点对某一消息进行投票，主节点合成投票意见并通知给其它节点。这些过程可以统一表示，并采用流水化来处理。

<img src ="/images/2019/hotstuff_3.png"/>

如果每个内容都必须经过二轮投票/三个阶段才能达成共识，如果有 m 个内容就需要执行 2m 次投票。管线设计(Pipelining)可以减少投票的次数，HotStuff 的基本思路如下：让每个节点在投第 i 轮的 prepare 阶段时，同时也是对其前一个内容 i-1 的 commit 阶段投票。这样做便可以节省对同一个内容重复投票的冗余，大幅提升效率。

##### 7.3.4 view change 优化

PBFT 中当发生 view change 时，需要节点对 view change 达成共识，然后节点把这个共识（以及已经达成了共识这件事）告诉新的 leader，新的 leader 还要把这个消息广播出去宣布 view change，于是 view change的复杂度是 $O(n^3)$ , 采用了聚合签名之后是 $O(n^2)$。这带来两个问题：

+ view change 的消息复杂度，
+ view change 必须要等到节点对于 view change 达成共识之后才会发生。

Hotstuff  把 PBFT 的两轮共识变成了三轮，然后借此把 view change 的复杂度变成了 O(n)。这个可以这么理解：传统的 view change是 $O(n^2)$ 消息复杂度，也就是说，所有的节点在 view change 之前会确认所有的节点确实都进行到下一个 view，而在 Hotstuff 中，view change 不需要等**我知道其他人也知道 view change 了**这件事就可以进行，于是，消息复杂度就降到了 $O(n)$，也就是说，只要诚实节点的内置 time out 到了，那么就可以发 view change 给新的 leader 开始 view change。

HotStuff 为什么需要把两轮变成三轮呢？上面提到的 BFT+Chain 结构的简化中，严格来说这两个通信复杂度为 $O(n)$ 的区块和 PBFT $O(n^2)$消息复杂度的 prepare 和 commit 还是有区别的，当有两个区块连起来的时候两边是相当的，但是其实每一个区块的消息复杂度都只有 $O(n)$，并不说明所有诚实节点都知道--**所有诚实节点都会达成共识**。而同样，view change 的消息复杂度也只有 O(n)，于是如果一条消息刚有第一个区块的时候view change 了，那么诚实节点会对于第一个区块是否达成了共识产生不一致，因为 prepare 和 view change看起来都很有道理。

而把两轮变成三轮之后我们就解决了这个问题。因为我们可以规定任何两轮之后的东西才是共识，而如果没有到两轮就不算--**对于 prepare 和 view change 都是如此**。于是，如果 view change 发生在第一轮之后，那么我们不认为之前 prepare 的是正确的，而 view change 也同理。相反，如果在第二轮之后发生 view change，那么由于已经经过了两轮，所以这条消息已经经过了 prepare，即便在 view change 之后也会最终达成共识。

##### 7.3.5 Pacemaker 机制

HotStuff 通过 pacemaker 机制从算法层面对共识安全（safety）和活性（liveness) 进行解耦合，将保证系统安全的部分抽离出来，然后将与具体应用相关的 heuristics 部分分离，来实现 liveness。

> HotStuff also implements a mechanism called *Pacemaker*, that guarantees liveness after GST. Pacemaker a) synchronizes all correct replicas and a unique leader into a common height for a sufficiently long period of time. It chooses the unique leader such that the correct replicas synchronized will make progress with the chosen leader. This mechanism decouples **liveness** from the protocol, which in turn decouples it from **safety**.

#### 7.4 总结

总体来说，Hotstuff 的核心思路如下：

1. 采用门限签名把每一轮的消息复杂度变成 $O(n)$。

2. 用 BFT+Chain 结构把 $O(n^2)$ 的共识变成了两轮 $O(n)$ 消息复杂度的区块提交。

3. 在这种结构下，把 view change 的消息复杂度降到 $O(n)$，然后为了防止 view change 造成的不一致，把两轮区块提交变成了三轮。
4. 共识过程采用流水化处理。

### 8. LibraBFT



#### 8.1 LibraBFT 简介

LibraBFT[^5] 基于 HotStuff[^4] ，进一步完善了 HotStuff 协议，LibraBFT 在 HotStuff的基础上引入显示的活跃机制并提供了具体的延时分析。LibraBFT在一个有全局统一时间（GST），并且网络最大延时（ΔT）可控的 Partial Synchrony 的网络中是有效的。并且，LibraBFT在所有验证节点都重启的情况下，也能够保证网络的一致性。

>LibraBFT is ==based on HotStuff==, a recent protocol that leverages several decades of scientific advances in Byzantine fault tolerance (BFT) and achieves the strong scalability and security properties required by internet settings. 

LibraBFT 每一轮(round)共识都会选举出一个 leader 节点，然后由 leader 节点发起提案(proposals)，收集投票(vote)，最后达成共识。在 LibraBFT 中，所有参与共识的节点称之为 validator，即验证节点。

LibraBFT 是变体的 HotStuff chain(这个链不是 block chain ,而是用于共识的 hash 链),在 HotStuff 的每轮(round)共识流程中，所有的信息交互都只和 leader 节点进行，然后 leader 节点的提案会以一条加密的 hash链组成(HotStuff chain)。在每轮共识中，被选举出的 leader 节点会基于节点自身最长的 HotStuff chain 提出 block 提案，如果提案是有效并且及时，剩余的诚实节点会使用自身的私钥签名该提案，并将通过的 vote 发送回 leader节点。当 leader 节点收集到足够的 (Quorum vote) 时，会将 vote 聚合成 Quorum证书(Quorum Certificate，QC)，当然，leader 会基于上述已延伸的最长 HotStuff chain 继续追加 QC，换言之，一轮共识成功后，leader 节点会基于自身最长的 HotStuff chain，按序追加提案 block(缩写B) 以及 QC(缩写C)，然后在将 QC 再次广播至剩余的节点，并开启下一轮的共识。当然这是共识成功的情况。在异常情况下，无论任何原因，如果当前 leader 节点没法及时共识成功，共识参与者会一直等到当前 round 的超时时间，然后在发起下一轮的共识。

 最后，如果足够的 blocks 以及 QCs 都能连续及时的通过共识，并且其中一个 block 达到了 commit 条件，那么，该 block 会被 commit ，换言之，该 block 以及 block 中 transaction 集合都会被落盘，并被确认。

<img src ="/images/2019/librabft_0.png"/>

                                                                                       HotStuff Chain

 上图描述的 HotStuff Chain，其可以表现为： 【ℎinit ← 𝐵1 ← 𝐶1 ← 𝐵2 ← 𝐶2 … ← 𝐵𝑛+1 ← 𝐶𝑛+1】



#### 8.2 LibraBFT 共识流程

根据论文描述 LibraBFT 流程如下图：

<img src ="/images/2019/librabft_1.png"/>

                 Overview of the LibraBFT protocol (simplified, excluding round synchronization)																		

**简单分析： **

1. round 1 的 leader node3 出了 block B1，广播消息到所有 nodes。
2. 其它 node 同意，回传 vote V1。
3. node3 收集投票 (包含自己的 vote)，形成 C1，传给所有 nodes。
4. (3 和 1 之间的间隔时间) node0 是 round 2 的 leader，收集其它 nodes 的状态，确定有 `N — f` 个 nodes 进入 round 2 后，产生 B2，回到步骤 1 ~ 3。

**详细分析：**

通过 round 3 来详细分析共识流程：

1. node1 被选举为 leader 节点，然后 node1 会基于自身最长链尾部 C2 发起 B3 提案，然后将提案广播至剩余节点(node0，2，3)，广播完成后，node1 会执行 B3 中的 transaction 列表，得到 execute state；

2. 当剩余节点(node0，2，3)收到 B3提案后，会执行 B3 中的交易，然后将 execute state 打包到 vote 中，在将其签名，并发送给 node1;

3. 一起都正常情况下，node1 会收到自身以外的 vote 请求，当收集到足够多的 vote 时，并且 vote 验证通过,包括 execute state 、签名、round 等，leader 节点会将 vote 集合打包成 quorum certificate(QC)，并将 QC 广播至剩余节点，到这里，一轮共识完毕。     

**block 提交**
共识轮次(round)用 int 表示，并且只会递增。在每一轮共识轮次中，都仅由一个 leader 节点发起一次提案，轮次仅在共识成功后或当前共识轮次超时才会结束，然后轮次自增1，启动下轮次共识。因此，假设 round(Bi) 为提案 Bi 的当前轮次值，并且，依据递增的规定可以得到 round(𝐵𝑖) < round(𝐵𝑖+1)；另外，如果 round(𝐵𝑖) + 1 = round(𝐵𝑖+1)，我们则称之为连续共识成功两轮，代表在这两轮共识中，都没有发生超时，并且共识都成功。如果同时满足以下两个条件：

- 𝐵1 ← 𝐶1 ← 𝐵2 ← 𝐶2 ← 𝐵3 ← 𝐶3
- round(𝐵3) = round(𝐵2) + 1 = round(𝐵1) + 2

那么，B1 则会被 commit，换言之，连续三个共识轮次成功的情况下，第一个共识的 block 就会被提交。总的来说，相对于 PBFT 的三阶段提交来说，LibraBFT 的只需1.5阶段，既可完成一轮共识，上述的 round(B3) 完成后，round(B1) 才会被 commit 相当于 PBFT 的 commit 阶段。其实这就是 HotStuff 中的做法。



#### 8.3 Data Synchronization



Leader 在提案以前，会先确保足够人数进入同一 round 才会出块。nodes 之间会定时互相传播自己的状态，然后向別人取得缺少的部份。同步的流程为:

1. 广播 DataSyncNotaification。
2. 依其它人的状态，向对方发送 DataSyncRequest。
3. 收到 DataSyncRequest 后，回复 DataSyncResponse。

目前的实现，request 一次传自己全部的状态，response 一次回复全部缺少的部份。详细实现如下:

<img src ="/images/2019/librabft_2.png"/>

目前代码跟白皮书上的伪代码有出入。



#### 8.4 Voting Constraint

符号说明：

- B 表示 区块(block)。
- C 表示 (法定证书)quorum certificate。
- B ← C 表示 C 的上一个 record 指向 B，即 C 有记录 B 的 hash。
- round(B) 表示 B 的 round 是多少。
- 对 B’ ← C’ ← B 来说，previous_round(B) 表示 round(B’)

除了只能投目前 round 的 block 以外，nodes 投票时还要遵守两个规则:

- First voting constraint: 投过 round x 的票后只能投 round y (y > x) 的票。
- Second voting constraint: 看过 2-chain B0 ← C0 ← B1 ← C1 后，收到新的 block B，previous_round(B) ≥ round(B0)

第一个规则符合自觉，只能投更新的 block。若同一 round 里，leader 不守规则产生两个不同的 block，voter 只会投看到的第一个，不会 double vote。

第二个规则表示有看到两个连续通过的 block 后，新的 block B 至少要指向 B0 或 round 数比它大的 block。注意，这里沒有要求 B 所在的 chain 上面一定有 B0。

#### 8.5 Commit Rule

对 3-chain B0 ← C0 ← B1 ← C1 ← B2 ← C2 且 round(B2) = round(B1)+1 = round(B0)+2 来说，B0 以及它之前的 records 的状态称为 committed，一但 committed，表示记录就不会掉了。

QC 有个选择性的栏位 commitment，用来记录最新的 committed state。以这里的例子来说，C2 的 commitment 会记录 state(C0)。

只管来说，若 node A 看到上述的 B0 ← … ← C2，表示 A 知道有 `N — f` 的 nodes 知道 B0 ← C0 ← B1 ← C1。所以之后通过的 block B，会满足 previous_round(B) ≥ round(B0)。表示 B 所在的 chain 上必定包含 B0。

#### 8.6 Punishment

有了 voting constraint 和 commit rule，大家可以举报谁不遵守规则，验证后可惩罚 (例如沒收 stake-in 的 token)。

#### 8.7 Network

文中提到在 P2P synchronization protocol 之上有加一层 gossip overlay，在 broadcast 的时候会随机传給 K (≤N) 个 nodes，然后通过收到的 nodes 继续传播消息。不过为简化讨论，假设 K = N。

考虑到网络状态会变化，timeout 的值由以下公式确定:

```rust
duration(n) = delta * (n - nc - 2)^r
```

- n 表示目前的 round。
- nc 表示最后一个 committed block 的 round。
- delta 和 r 是常数。

直观来说，愈久没有新的 committed block，timeout 时间愈久。假设无法同意新的 block 是因为网络状态不佳，逐渐加长 timeout 时间合理。但若是因为 byzantine node (拜占庭节点) 造成的，反而让 byzantine node 拖更久没有 liveness。实际情况依「拜占庭节点和网络状态不佳」何者更容易发生，来决定参数怎么设置。

#### 8.8 总结

We have presented LibraBFT, a state machine replication system based on the HotStuff protocol [5]
and designed for the Libra Blockchain [2]. LibraBFT provides safety and liveness in a Byzantine
setting when up to one-third of voting rights are held by malicious actors, assuming that the network is partially synchronous. In this report, we have presented detailed proofs of safety and liveness and covered many important practical considerations, such as networking and data structures. We have shown that LibraBFT is compatible with proof of stake and can generate incentives for a variety of behaviors, such as proposing blocks and voting. Thanks to the simplicity of the safety argument in LibraBFT, we also provided criteria to detect malicious attempts to break safety. These criteria will be instrumental for the progressive migration of the Libra infrastructure to a permissionless model.

#### 8.9 Future work

This report constitutes an initial proposal for LibraBFT and is meant to be updated in the future. In the next version, we intend to share the code for our reference implementation in a simulated environment and provide experimental results, both using this simulation and using the production implementation currently developed by Calibra engineers.

In the future, we would like to improve our theoretical analysis in several ways. We plan to make
our networking assumptions more precise, with additional studies on message sizes and probabilistic gossiping. Regarding the integration of LibraBFT with the Libra Blockchain, we would like to cover fairness and discuss how light clients can authenticate the set of validators for each epoch. Economic incentives should reward additional positive behaviors, such as creating timeouts, and specifications should provide an external protocol for auditors to report violations of safety rules.

On a practical level, we have not yet analyzed resource consumption (memory, CPU, etc.) in the
presence of malicious participants. Heuristics for leader selection, a precise description of the VRF
solution, and possibly adaptive policies will likely be required to increase the robustness of the system in case of malicious leaders or targeted attacks on leaders.

In the long term, we hope that our efforts on precise specifications and detailed proofs will pave the
way for mechanized proofs of safety and liveness of LibraBFT.



### 9. 参考资料

+ [HotStuff: BFT Consensus in the Lens of Blockchain](https://arxiv.org/pdf/1803.05069.pdf)
+ [State Machine Replication in the Libra Blockchain](https://developers.libra.org/docs/papers/libra-consensus-state-machine-replication-in-the-libra-blockchain.pdf)
+ [Libra 采用的 HotStuff 算法作者亲述：「尤物」诞生记](https://zhuanlan.zhihu.com/p/72776441)
+ [一文简述 HotStuff 的工作原理](https://www.chainnews.com/articles/215569914405.htm)
+ [LibraBFT算法简述](https://bbs.vechainworld.io/topic/200/librabft算法简述)





[^1]: https://en.wikipedia.org/wiki/Two_Generals%27_Problem](https://en.wikipedia.org/wiki/Two_Generals'_Problem
[^2]: https://dl.acm.org/citation.cfm%3Fid%3D357176
[^3]: http://pmg.csail.mit.edu/papers/osdi99.pdf
[^4]: M. Yin, D. Malkhi, M. K. Reiterand, G. G. Gueta, and I. Abraham, “HotStuff: BFT consensus

in the lens of blockchain,” 2019. https://arxiv.org/pdf/1803.05069.pdf

[^5]: State Machine Replication in the Libra Blockchain https://developers.libra.org/docs/papers/libra-consensus-state-machine-replication-in-the-libra-blockchain.pdf
[^6]: Impossibility of Distributed Consensus with One Faulty Process  https://groups.csail.mit.edu/tds/papers/Lynch/jacm85.pdf
[^7]: https://en.wikipedia.org/wiki/Clock_drift
[^8]: https://en.wikipedia.org/wiki/Clock_skew
[^9]: https://en.wikipedia.org/wiki/Network_Time_Protocol
[^10]: https://en.wikipedia.org/wiki/Real-time_clock