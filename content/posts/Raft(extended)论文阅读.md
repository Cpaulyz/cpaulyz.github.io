+++

title = "Raft(extended)论文阅读"

date = "2022-01-29"

tags = [
    "MIT6.824",
    "distributed system",
    "paper",

]

+++

# Raft(extended)论文阅读

> 论文：http://nil.csail.mit.edu/6.824/2020/papers/raft-extended.pdf
>
> 翻译：https://www.cnblogs.com/linbingdong/p/6442673.html
>
> 参考1：https://tanxinyu.work/raft/
>
> 参考2：https://mp.weixin.qq.com/s?__biz=MzIwODA2NjIxOA==&mid=2247484140&idx=1&sn=37876b5dda5294ea7f6211f0a3300ea5&chksm=97098129a07e083fe65f8b87c2ec516b630a8f210961038f0091fbcd69468b41edbe193891ee&scene=21#wechat_redirect

## 摘要

Raft是一种共识算法，和Paxos等效，但有着不同的结构

* Raft比Paxos更容易理解
  * 拆分共识算法的关键要素，如领导人选举、日志复制和安全，并加强一致性，以减少必须考虑的状态数量
* 为构建实用的系统提供了更好的基础
* Raft 还包括一个用于变更集群成员的新机制，它使用重叠的大多数（overlapping majorities）来保证安全性。

## 1 Intruduction

Raft 在许多方面类似于现有的一致性算法（尤其是 Oki 和 Liskov 的 Viewstamped Replication [29,22]），但它有几个新特性：

- **Strong leader**：在 Raft 中，日志条目（log entries）只从 leader 流向其他服务器。 这简化了复制日志的管理，使得 raft 更容易理解。
- **Leader 选举**：Raft 使用随机计时器进行 leader 选举。 这只需在任何一致性算法都需要的心跳（heartbeats）上增加少量机制，同时能够简单快速地解决冲突。
- **成员变更**：Raft 使用了一种新的联合一致性方法，其中两个不同配置的大多数在过渡期间重叠。 这允许集群在配置更改期间继续正常运行。

论文结构：

* 复制状态机问题（第 2 节）
* Paxos 的优点和缺点（第3节）
* 描述了我们实现易理解性的方法（第 4 节）
* 提出了 Raft 一致性算法（第 5-8 节）
* 评估 Raft（第 9 节）
* 相关工作（第 10 节）。

## 2 复制状态机

* **原理**：任何初始状态一样的状态机，如果执行的命令序列一样，则最终达到的状态也一样。如果将此特性应用在多参与者进行协商共识上，可以理解为系统中存在多个具有完全相同的状态机（参与者），这些状态机能最终保持一致的关键就是起始状态完全一致和执行命令序列完全一致。

* **作用**：解决分布式系统中的各种容错问题，例如一组服务器上存在状态机计算相同状态的相同副本，并且即使某些服务器宕机，也可以继续运行。

* **实现**：通常使用复制日志实现。每个服务器存储一个包含一系列命令的日志，每个日志中命令都相同并且顺序也一样，因此每个状态机处理相同的命令序列，从而能得到相同的状态和相同的输出序列。

<img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220129150357888.png" alt="image-20220129150357888" style="zoom:50%;" />

用于保证复制日志的一致性的算法就叫做共识算法，共识算法就是用来确保每个节点上的日志顺序都是一致的。

每个服务器上的共识模块会与其他服务器上的共识模块通信， 一旦命令被正确复制，每个服务器上的状态机按日志顺序处理它们，并将输出返回给客户端。 这样就形成了高可用的复制状态机。

实际系统中的共识算法通常具有以下属性：

- 它们确保在所有非拜占庭条件下（包括网络延迟，分区和数据包丢失，重复和乱序）的安全性（不会返回不正确的结果）。
- 只要任何大多数（过半）服务器都可以运行，并且可以相互通信和与客户通信，一致性算法就可用。 因此，五台服务器的典型集群可以容忍任何两台服务器的故障。 假设服务器突然宕机，它们可以稍后从状态恢复并重新加入群集。
- 它们不依赖于时序来确保日志的一致性：错误的时钟和极端消息延迟在最坏的情况下会导致可用性问题，但可以保证一致性。
- 在通常情况下，只要集群的大部分（过半服务器）已经响应了单轮远程过程调用，命令就可以完成; 少数（一半以下）慢服务器不会影响整个系统性能。

## 3 Paxos存在的问题

paxos存在两个巨大的缺点

* 难以理解
  * 作者选择了single-degree Paxos作为基础，Single-decree Paxos 分成两个阶段，这两个阶段没有简单直观的说明，并且不能被单独理解。因此，很难理解为什么该算法能起作用。
  * Multi-Paxos 的合成规则又增加了许多复杂性
* 不能为构建实际的实现提供良好的基础
  * 没有针对 multi-Paxos 的广泛同意的算法
  * 几乎所有的实现都是从 Paxos 开始，然后发现很多实现上的难题，接着就开发了一种和 Paxos 完全不一样的架构。这样既费时又容易出错
  * “在Paxos算法描述和实现现实系统之间有着巨大的鸿沟。最终的系统往往建立在一个还未被证明的协议之上。”

## 4 为可理解性而设计

最大目标：可理解性

使用了两个技术

* 分解为子问题
  * 例如，Raft 算法被我们分成 leader 选举，日志复制，安全性和成员变更几个部分。
* 减少状态的数量来简化状态空间
  * 所有的日志是不允许有空洞的，并且 Raft 限制了使日志之间不一致的方式。
  * 使用随机化来简化 Raft 中的 leader 选举算法。

## 5 Raft

![image-20220201161853907](https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220201161853907.png)

保证：

- `Election Safety`：每个 term 最多只会有一个 leader；集群同时最多只会有一个可以读写的 leader。
- `Leader Append-Only`：leader 的日志是只增的。
- `Log Matching`：如果两个节点的日志中有两个 entry 有相同的 index 和 term，那么它们就是相同的 entry。
- `Leader Completeness`：一旦一个操作被提交了，那么在之后的 term 中，该操作都会存在于日志中。
- `State Machine Safety`：一致性，一旦一个节点应用了某个 index 的 entry 到状态机，那么其他所有节点应用的该 index 的操作都是一致的

### 5.1 Raft基础

#### 5.1.1 节点

在任何时刻，每一个服务器节点都处于这三个状态之一：

- `Leader`：集群内最多只会有一个 leader，负责发起心跳，响应客户端，创建日志，同步日志。
- `Candidate`：leader 选举过程中的临时角色，由 follower 转化而来，发起投票参与竞选。
- `Follower`：接受 leader 的心跳和日志同步数据，投票给 candidate。Follower都是被动的，他们不会发送任何请求，只是简单的响应来自 leader 和 candidate 的请求

<img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220129185156907.png" alt="image-20220129185156907" style="zoom:50%;" />

#### 5.1.2 任期Term

* Raft 把时间分割成任意长度的任期（term）。
* 任期用连续的整数标记。每一段任期从一次选举开始，一个或者多个 candidate 尝试成为 leader 。
* 如果一个 candidate 赢得选举，然后他就在该任期剩下的时间里充当 leader 。
* 在某些情况下，一次选举无法选出 leader 。在这种情况下，这一任期会以没有 leader 结束；一个新的任期（包含一次新的选举）会很快重新开始。
* Raft 保证了在任意一个任期内，最多只有一个 leader 。

<img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220129185454952.png" alt="image-20220129185454952" style="zoom:50%;" />

term在raft中充当逻辑时钟的角色，每一个服务器节点存储一个当前任期号，服务器之间通信的时候会交换当前任期号：

* 如果一个服务器的当前任期号比其他的小，该服务器会将自己的任期号更新为较大的那个值。
* 如果一个 candidate 或者 leader 发现自己的任期号过期了，它会立即回到 follower 状态。
* 如果一个节点接收到一个包含过期的任期号的请求，它会直接拒绝这个请求。

#### 5.1.3 RPC

Raft 算法中服务器节点之间使用 RPC 进行通信，并且基本的一致性算法只需要两种类型的 RPC。

* 请求投票（RequestVote） RPC 由 candidate 在选举期间发起（章节 5.2）
* 追加条目（AppendEntries）RPC 由 leader 发起，用来复制日志和提供一种心跳机制（章节 5.3）

第 7 节为了在服务器之间传输快照增加了第三种 RPC。当服务器没有及时的收到 RPC 的响应时，会进行重试， 并且他们能够并行的发起 RPC 来获得最佳的性能。

### 5.2 Leader选举

**选举时机：**如果一个 follower 在一段选举超时时间内没有接收到任何消息，它就假设系统中没有可用的 leader ，然后开始进行选举以选出新的 leader 。

> 当服务器程序启动时，他们都是 follower 。一个服务器节点只要能从 leader 或 candidate 处接收到有效的 RPC 就一直保持 follower 状态。Leader 周期性地向所有 follower 发送心跳（不包含日志条目的 AppendEntries RPC）来维持自己的地位。

**选举过程**：follower 增加自己的当前任期号并且转换到 candidate 状态，*投票给自己*并且并行地向集群中的其他服务器节点发送 RequestVote RPC（让其他服务器节点投票给它）。Candidate 会一直保持当前状态直到以下三件事情之一发生：

* (a) 赢得选举（收到过半的投票）
  * 当一个 candidate 获得过半投票，就赢得了这次选举并成为 leader 。
  * 对于同一个任期，每个服务器节点只会投给一个 candidate ，按照先来先服务（first-come-first-served）的原则（5.4 节在投票上增加了额外的限制）。
  * 一旦 candidate 赢得选举，就立即成为 leader 。然后它会向其他的服务器节点发送心跳消息来确定自己的地位并阻止新的选举。
* (b) 其他的服务器节点成为 leader 
  * candidate 可能会收到另一个声称自己是 leader 的服务器节点发来的 AppendEntries RPC 
  * 如果这个 leader 的任期号（包含在RPC中）不小于 candidate 当前的任期号，那么 candidate 会承认该 leader 的合法地位并回到 follower 状态。 
  * 如果 RPC 中的任期号比自己的小，那么 candidate 就会拒绝这次的 RPC 并且继续保持 candidate 状态。
* (c) 一段时间之后没有任何获胜者
  * 有多个 follower 同时成为 candidate 来瓜分选票，因此raft算法设定了一个**随机的选举超时时间**
  * 一旦某个candidate选举超时，它就会增加当前任期号来开始一轮新的选举，
  * 因为超时时间是随机的，大概率只有一个节点会超时，然后该节点赢得选举并在其他节点超时之前发送心跳，抢先成为leader
  * 注意，**每次收到AppendEntries或是超时后重设的electionTimeout都是随机的**，而不是在一开始启动的时候随机选择一个数字后就不再改变。

> Leader赢得选举后，应该马上发送一条空的AppendEntries来宣誓自己的地位

### 5.3 日志复制

> 引入了entry和log的概念。
>
> - `entry`：Raft 中，将每一个事件都称为一个 entry，每一个 entry 都有一个表明它在 log 中位置的 index（之所以从 1 开始是为了方便 `prevLogIndex` 从 0 开始）。只有 leader 可以创建 entry。entry 的内容为 `<term, index, cmd>`，其中 cmd 是可以应用到状态机的操作。在 raft 组大部分节点都接收这条 entry 后，entry 可以被称为是 committed 的。
> - `log`：由 entry 构成的数组，只有 leader 可以改变其他节点的 log。 entry 总是先被 leader 添加进本地的 log 数组中去，然后才发起共识请求，获得 quorum 同意后才会被 leader 提交给状态机。follower 只能从 leader 获取新日志和当前的 commitIndex，然后应用对应的 entry 到自己的状态机。
>
> <img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220129223941102.png" alt="image-20220129223941102" style="zoom:50%;" />

客户端请求 ➡️ Leader 把该指令作为一个新的条目追加到日志中去 ➡️ 并行的发起 AppendEntries RPC 给其他的服务器 ➡️ follower复制该条目 ➡️ 当该条目被过半服务器 ➡️ leader commit该条目并在状态机执行该指令 ➡️ 把执行的结果返回给客户端

前文中，我们提到Raft保证两个性质：

* 如果不同日志中的两个条目拥有相同的索引和任期号，那么他们存储了相同的指令。
* 如果不同日志中的两个条目拥有相同的索引和任期号，那么他们之前的所有日志条目也都相同。

第一个性质通过leader的append only来保证

第二个性质由 AppendEntries RPC 执行一个简单的一致性检查所保证的

* Leader 针对每一个 follower 都维护了一个 nextIndex ，表示 leader 要发送给 follower 的下一个日志条目的索引。
* 新 leader 时，该 leader 将所有 nextIndex 的值都初始化为自己最后一个日志条目的 index 加1。
* 在发送 AppendEntries RPC 的时候，leader 会将前一个日志条目的索引位置（也就是nextIndex-1）和任期号包含在里面。
* 如果 follower 在它的日志中找不到包含相同索引位置和任期号的条目，那么他就会拒绝该新的日志条目，leader会减少nextIndex并重试appendEntries RPC
* 最终 nextIndex 会在某个位置使得 leader 和 follower 的日志达成一致。此时，AppendEntries RPC 就会成功，将 follower 中跟 leader 冲突的日志条目全部删除然后追加 leader 中的日志条目（如果有需要追加的日志条目的话），使follower和leader中的日志条目达成一致

> 问题：为什么需要一致性检查？因为有些时候 follower 的日志可能和新的 leader 的日志不同，Follower 可能缺少一些在新 leader 中有的日志条目，也可能拥有一些新 leader 没有的日志条目，或者同时发生。缺失或多出日志条目的情况可能会涉及到多个任期。
>
> <img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220129224619229.png" alt="image-20220129224619229" style="zoom:50%;" />
>
> > 当一个 leader 成功当选时（最上面那条日志），follower 可能是（a-f）中的任何情况。
> >
> > Follower 可能会缺少一些日志条目（a-b），可能会有一些未被提交的日志条目（c-d），或者两种情况都存在（e-f）。例如，场景 f 可能这样发生，f 对应的服务器在任期 2 的时候是 leader ，追加了一些日志条目到自己的日志中，一条都还没提交（commit）就崩溃了；该服务器很快重启，在任期 3 重新被选为 leader，又追加了一些日志条目到自己的日志中；在这些任期 2 和任期 3 中的日志都还没被提交之前，该服务器又宕机了，并且在接下来的几个任期里一直处于宕机状态。

### 5.4 安全性

> 5.4主要保证前文提到的`Leader Completeness`，即一旦一个操作被提交了，那么在之后的 term 中，该操作都会存在于日志中。

仅考虑5.2和5.3，可能出现异常情况：一个 follower 可能会进入不可用状态，在此期间，leader 可能提交了若干的日志条目，然后这个 follower 可能会被选举为 leader 并且用新的日志条目覆盖这些日志条目；结果，不同的状态机可能会执行不同的指令序列。

因此，我们需要对leader选举增加一些额外的限制，**保证leader 都包含了之前各个任期所有被提交的日志条目。**

#### 5.4.1  选举限制

Raft中，日志条目的传送是单向的，只从 leader 到 follower，并且 leader 从不会覆盖本地日志中已经存在的条目。

我们需要保证：能赢得选举的candidate包含了所有已经提交的日志条目。

解决方案：RequestVote RPC 中包含了 candidate 的日志信息，如果投票者自己的日志比 candidate 的还新，它会拒绝掉该投票请求。

> “新”的定义：
>
> Raft 通过比较两份日志中**最后一条日志条目**的索引值和任期号来定义谁的日志比较新。
>
> * 如果两份日志最后条目的任期号不同，那么任期号大的日志更新。
> * 如果两份日志最后条目的任期号相同，那么日志较长的那个更新。

#### 5.4.2 之前任期的日志

一言以蔽之： **Leader 不能提交之前任期的日志，只能通过提交自己任期的日志，从而间接提交之前任期的日志。**

一篇非常详细的解读：https://mp.weixin.qq.com/s/jzx05Q781ytMXrZ2wrm2Vg

> 思考：为何Raft算法在进行投票时，不对CommitIndex进行比较呢？
>
> （用例来源：http://vearne.cc/archives/1510）
>
> 个人认为可能存在这样一种边界条件
>
> <img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220131132501716.png" alt="image-20220131132501716" style="zoom:50%;" />
>
> commitIndex应该是3，但是在s2、s3更新commitIndex到3前，s1、s2宕机了
>
> <img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220131132554232.png" alt="image-20220131132554232" style="zoom:50%;" />
>
> 此时s3、s4的commitIndex可能都是2，如果比较CommitIndex的话s4可能当选。但正确的结果应该是s3当选，这样才能保证理应commit的（index=3，term=1）最终在s3当选以后会被复制到s4、s5以后commit

#### 5.4.3 安全性论证

这一部分使用反证法来证明Leader Completeness

<img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220131133350193.png" alt="image-20220131133350193" style="zoom:50%;" />

假设 leader 完整性特性是不满足的，然后我们推出矛盾来。假设任期 T 的 leader（leader T）在任期内提交了一个日志条目，但是该日志条目没有被存储到未来某些任期的 leader 中。假设 U 是大于 T 的没有存储该日志条目的最小任期号。

1. U 一定在刚成为 leader 的时候就没有那条被提交的日志条目了（leader 从不会删除或者覆盖任何条目）。
2. Leader T 复制该日志条目给集群中的过半节点，同时，leader U 从集群中的过半节点赢得了选票。因此，至少有一个节点（投票者）同时接受了来自 leader T 的日志条目和给 leader U 投票了，如图 9。该投票者是产生矛盾的关键。
3. 该投票者必须在给 leader U 投票之前先接受了从 leader T 发来的已经被提交的日志条目；否则它就会拒绝来自 leader T 的 AppendEntries 请求（因为此时它的任期号会比 T 大）。
4. 该投票者在给 leader U 投票时依然保有这该日志条目，因为任何 U 、T 之间的 leader 都包含该日志条目（根据上述的假设），leader 从不会删除条目，并且 follower 只有跟 leader 冲突的时候才会删除条目。
5. 该投票者把自己选票投给 leader U 时，leader U 的日志必须至少和投票者的一样新。这就导致了以下两个矛盾之一。
6. 首先，如果该投票者和 leader U 的最后一个日志条目的任期号相同，那么 leader U 的日志至少和该投票者的一样长，所以 leader U 的日志一定包含该投票者日志中的所有日志条目。这是一个矛盾，因为该投票者包含了该已被提交的日志条目，但是在上述的假设里，leader U 是不包含的。
7. 否则，leader U 的最后一个日志条目的任期号就必须比该投票者的大了。此外，该任期号也比 T 大，因为该投票者的最后一个日志条目的任期号至少和 T 一样大（它包含了来自任期 T 的已提交的日志）。创建了 leader U 最后一个日志条目的之前的 leader 一定已经包含了该已被提交的日志条目（根据上述假设，leader U 是第一个不包含该日志条目的 leader）。所以，根据日志匹配特性，leader U 一定也包含该已被提交的日志条目，这里产生了矛盾。
8. 因此，所有比 T 大的任期的 leader 一定都包含了任期 T 中提交的所有日志条目。
9. 日志匹配特性保证了未来的 leader 也会包含被间接提交的日志条目，例如图 8 (d) 中的索引 2。

### 5.5 Follower和Candidate崩溃

之前我们只关注了leader崩溃的情况

如果 follower 或者 candidate 崩溃了，那么后续发送给他们的 RequestVote 和 AppendEntries RPCs 都会失败。Raft 通过无限的重试来处理这种失败；如果崩溃的机器重启了，那么这些 RPC 就会成功地完成。

这要求 Raft 的 RPC 幂等，来保证这样的重试不会造成任何伤害。例如，一个 follower 如果收到 AppendEntries 请求但是它的日志中已经包含了这些日志条目，它就会直接忽略这个新的请求中的这些日志条目。

### 5.6 定时（timing）和可用性

只要整个系统满足下面的时间要求，Raft 就可以选举出并维持一个稳定的 leader：

> 广播时间（broadcastTime） << 选举超时时间（electionTimeout） << 平均故障间隔时间（MTBF）
>
> 其中，广播时间指的是一个服务器并行地发送 RPCs 给集群中所有的其他服务器并接收到响应的平均时间；选举超时时间就是在 5.2 节中介绍的选举超时时间；平均故障间隔时间就是对于一台服务器而言，两次故障间隔时间的平均值。

* 广播时间（broadcastTime） << 选举超时时间（electionTimeout）
  * 这样 leader 才能够可靠地发送心跳消息来阻止 follower 开始进入选举状态；
  * 再加上随机化选举超时时间的方法，这个不等式也使得选票瓜分的情况变得不可能
* 选举超时时间（electionTimeout） << 平均故障间隔时间（MTBF）
  * 当 leader 崩溃后，整个系统会有大约选举超时时间不可用；我们希望该情况在整个时间里只占一小部分。

## 6 集群成员变更

实践中，我们可能会需要：

* 替换机器
* 更改集群复制度（副本数）

并希望能够实现配置变更自动化，即不需要重启整个集群，也不需要手工参与。

### 实现

在 Raft 中，集群先切换到一个过渡的配置，我们称之为**联合一致（joint consensus）**

joint consensus 指的是包含新／旧配置文件全部节点的中间状态：

- entries 会被复制到新／旧配置文件中的所有节点；
- 新／旧配置文件中的任何一个节点都有可能被选为 leader；
- 共识（选举或提交）需要同时在新／旧配置文件中分别获取到多数同意（separate majorities）

（注：separate majorities 的意思是需要新／旧集群中的多数都同意。比如如果是从 3 节点切换为全新的 9 节点， 那么要求旧配置中的 2 个节点，和新配置中的 5 个节点都同意，才被认为达成了一次共识）

也就是说，在一次配置变更中，一共有三个状态：

- `C-old`：使用旧的配置文件；
- `C-old,new`：同时使用新旧配置文件，也就是新／旧节点的并集；
- `C-new`：使用新的配置文件。

其中，配置文件的变更用特殊的log entry实现，一旦某个服务器将该新配置日志条目增加到自己的日志中，它就会用该配置来做出未来所有的决策（无论该配置日志是否已经被提交）。这就意味着 leader 会使用 `C-old,new` 的规则来决定` C-old,new` 的日志条目是什么时候被提交的-

<img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220205171543631.png" alt="image-20220205171543631" style="zoom:50%;" />

图中有几个关键的时间点：

1. `C-old,new` 被创建，集群进入 joint consensus，
2. `C-old,new `被 committed，`C-old `已经不再可能被选为 leader；
3. leader 创建并传播 `C-new`；
4. `C-new` 被committed，此后不在 `C-new` 内的节点不允许被选为 leader，如有 leader 不在 `C-new` 则自行退位。

### 引入问题

**第一个问题**：新的服务器开始时可能没有存储任何的日志条目。当这些服务器以这种状态加入到集群中，它们需要一段时间来更新来赶上其他服务器，这段它们无法提交新的日志条目。

* Raft 在配置变更前引入了一个额外的阶段，在该阶段，新的服务器以没有投票权身份加入到集群中来（leader 也复制日志给它们，但是考虑过半的时候不用考虑它们）。一旦该新的服务器追赶上了集群中的其他机器，配置变更就可以按上面描述的方式进行。

**第二个问题：**集群的 leader 可能不是新配置中的一员。

* leader 一旦提交了 C-new 日志条目就会退位（回到 follower 状态）

**第三个问题：**那些被移除的服务器（不在 C-new 中）可能会扰乱集群。这些服务器将不会再接收到心跳，所以当选举超时，它们就会进行新的选举过程。它们会发送带有新任期号的 RequestVote RPCs ，这样会导致当前的 leader 回到 follower 状态。新的 leader 最终会被选出来，但是被移除的服务器将会再次超时，然后这个过程会再次重复，导致系统可用性很差。

* 方案一：当服务器在最小选举超时时间内收到一个 RequestVote RPC，它不会更新任期号或者投票。
* 方案二：[预投票](https://tanxinyu.work/raft/#%E9%A2%84%E6%8A%95%E7%A5%A8)

## 7 日志压缩

无限制增长的日志会对系统性能造成影响：

* 占用越来越多的空间
* 需要花更多的时间来回放

使用 snapshot 技术可以对日志进行压缩

<img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220205172948584.png" alt="image-20220205172948584" style="zoom:50%;" />

快照内容包括：

- 状态机当前的状态。
- 状态机最后一条应用的 entry 对应的 index 和 term。
- 集群最新配置信息。

服务器可以独立地创建快照。

当leader需要发送的日志index被快照了，直接通过InstallSnapshotRPC发送快照即可。

> follower 拿到非过期的 snapshot 之后直接覆盖本地所有状态即可，不需要留有部分 entry，也不会出现 snapshot 之后还存在有效的 entry。因此 follower 只需要判断 `InstallSnapshot RPC` 是否过期即可。过期则直接丢弃，否则直接替换全部状态即可。

<img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220205174902773.png" alt="image-20220205174902773" style="zoom:50%;" />

两个性能问题：

* 何时创建快照？
  * 一个简单的策略就是设置一个阈值，当日志大小达到一个固定大小的时候就创建一次快照。
* 如何创建快照？
  * 写时复制，copy-on-write

## 8 客户端交互

具体参考：https://cloud.tencent.com/developer/article/1746099

关于一致性模型的介绍：https://tanxinyu.work/consistency-and-consensus/