+++

title = "MIT6.824 Lecture3 GFS"

date = "2021-12-15"

tags = [
    "MIT6.824",
    "distributed system",
]

+++

# LEC3 GFS

https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-03-gfs

https://www.bilibili.com/video/BV1R7411t71W?p=3

## 论文阅读

> 推荐看这篇非常详细的论文解读：https://spongecaptain.cool/post/paper/googlefilesystem/

### 0 Abstract

GFS：a scalable distributed file syetem for large distributed data-intensive applications.

* fault tolerance
* high aggregate performance to a large number of clients

### 1 Introduction

* 目的：满足google快速增长的数据处理需求
* 与传统FS相同的目标：performance, scalability, reliability, availability
* 重新审视了传统文件系统的设计，提出了不同的观点：

1. 组件故障是常态，而不是异常。 -> 系统必须集成持续监控、错误检测、容错和自动恢复。
2. 以传统标准来度量，文件是巨大的。 -> 必须重新检查设计假设和参数，如I/O操作和块大小。
3. 大多数文件都是通过附加新数据而不是覆盖现有数据进行修改的。 -> 文件中的随机写入实际上不存在，很多时候一旦写了，文件就只能读，而且通常只能按顺序读。
4. 对于应用程序和文件系统API的协同设计可以提高系统的灵活性，从而使整个系统收益。 -> 例如，我们放松了GFS的一致性模型，大大简化了文件系统，而不会给应用程序带来沉重的负担。我们还引入atomic append操作，以便多个客户端可以同时append到一个文件，而无需它们之间进行额外的同步。

### 2 Design Overview

#### 2.1 Assumptions

更详细地进行一些假设：

* FS由许多廉价组件构成，它必须经常自我监控，并定期检测、容忍组件故障，并及时从中恢复。

* 系统存储了少量的大文件，几GB的文件很常见，需要进行有效的管理；而对于小文件，我们支持但不需要进行额外优化。

* **读工作负载**主要由两种读方式构成：大规模的串行读以及小规模的随机读

  - **大规模顺序读**：顺序（磁盘地址连续地）读取**数百及以上个** KB 大小的数据（或者单位改成 MB）；
  - **小规模随机读**：以任意偏移量读取几个 KB 大小的数据；

  > 小规模随机读会有优化，比如进行排序后的批处理化，以稳定地遍历文件（排序可能是按照索引的指针大小），而不是来回地随机读取。

* **写工作负载**主要是大规模的、连续（即串行的）的写操作，这些操作将数据追加到文件末尾。
  * 这要求：文件一旦写好，就几乎不会进行覆写
  * GFS 支持在文件的任意位置进行修改，但是并不会进行优化，存在并发安全问题，因此应当尽量避免使用。
* 系统需要支持并发写，即支持数百台机器并发地追加数据到一个文件。
  * 我们的文件常用于生产-消费环境，多个生产者并发append，同时还有消费者在读
* 高带宽比低延迟更重要。
  * 大多数的应用更在乎高速率地处理大量数据，但是很少应用对单个读写操作由严格的响应时间要求。
  * 参考 [StackOverflow](https://stackoverflow.com/questions/5350748/high-bandwidth-vs-low-latency) 相关问题的回答

#### 2.2 Interface

* 但是出于效率和使用性的角度，并没有实现标准的文件系统 POSIX API
* 常见的操作：create, delete, open, close, read and write
* 此外，GFS还支持：
  * snapshot: create a copy of a file or a directory tree at low cost
  * record append: allow multiple clients to append data to the same file concurrently while guaranteeingthe atomicity

#### 2.3 Architecture

![image-20211211155331510](https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20211211155331510.png)

* A GFS cluster consists of:

  * a single **master**

  * multiple **chunkservers**

  * accessed by multiple **clients**

* 以上都是运行在Linux上的用户级进程

* 文件被划分为许多固定大小的chunks

  * 每个chunk都被分配一个全局唯一且不可变的的64位chunk handle，由master根据chunk创建时间来分配
  * chunkservers在本地磁盘上存储chunks（和Linux files一样）
  * 为保证可靠性，每个chunk都在多个chunkservers上有副本，默认为3副本，用户可以根据命名空间指定不同的复制级别

* master维护了整个文件系统的元数据

  * 包括 namespace, access control information, the map-ping from files to chunks, and the current locations of chunks.

* master还控制全系统的活动

  * chunk lease management
  * garbage collection of orphaned chunks
  * chunk migration between chunkserver

* master通过hearbeat定期和chunkserver通信

* GFS clients 实现文件系统API来与master和chunkservers通信

  * 从master获取元数据
  * 数据承载通信都直接发送到chunkserver

* client和chunkserver都不缓存文件数据

  * client不缓存，因为文件很大，且流式读取，缓存没用，反而会导致一致性问题
  * 但是client缓存元数据
  * chunkserver不需要缓存，因为所有文件都是本地的linux file，把缓存交给操作系统即可

#### 2.4 Single Master

* 简化设计
* 使得master可以根据全局信息快速做出决策
* 尽量减少读写操作，以免成为瓶颈
* 一个简单的读操作如下：
  1. client根据固定的chunk size，将file offset换算成chunk index
  2. client向master请求，发送(filename, chunk index)，master返回元数据（包括chunk handle和replicas position），client将元数据缓存一段时间（从而减少很多针对同一个chunk块的client-master通信）
  3. client向其中一个副本发起请求（最有可能是最近的），发送(chunk handle, byte range)，得到结果

#### 2.5 Chunk Size

64MB，远远大于典型的单机文件系统 chunk 的大小

优点：

* 减少client-master的通信，因为对同一块进行多次读写仅仅需要向 Master 服务器发出一次初始请求，就能获取全部的块位置信息
* 减少了 GFS Client 与 GFS chunkserver 进行交互的数据开销，这是因为数据的读取具有连续读取的倾向，即读到 offset 的字节数据后，下一次读取有较大的概率读紧挨着 offset 数据的后续数据，chunk 的大尺寸相当于提供了一层缓存，减少了网络 I/O 的开销；
* 它减少了存储在主服务器上的元数据的大小。这允许我们将元数据保存在内存中。

缺点：

* 小数据量（比如仅仅占据一个 chunk 的文件，文件至少占据一个 chunk）的文件很多时，当很多 GFS Client 同时访问同一个文件时， 会造成局部的 hot spots 热点。

#### 2.6 Metadata

master 存储了三种主要的元数据，都存储在内存中

* the fileand chunk namespaces
* the mapping from files to chunks
* the locations of each chunk’s replicas

前两者会通过log的形式存储持久化到本地磁盘并备份到远程机器

* 方便变更
* 宕机时保证持久性

the locations of each chunk’s replicas不需要持久化，每次master启动的时候找chunkserver要就可以

* **In-memeory Data Sturcture**
  * 方便遍历表，进行周期性的扫描，以进行 chunk GC, re-replication in the presence of chunkserver fail-ures, and chunk migration to balance load and disk space usage across chunkservers
  * 如果需要支持更大的文件系统，在master加内存就好，和GFS提供的简单性、可靠性、性能和灵活性相比是一个很小的代价
* **Chunk Locations**
  * 不需要持久化，在启动的时候轮询chunkserver获取即可
  * 通过心跳信息来保持更新，并监控chunkserver的状态
  * chunkserver 对它自己的磁盘上有什么块或没有什么块有最终决定权。试图在 Master 服务器上维护这个信息的一致视图是没有意义的，因为 chunkserver 上的错误可能会导致块自动消失(例如，磁盘可能坏了并被禁用)，或者运维为 chunkserver 重命名。
* **Operation Log**
  * 它是 GFS 系统的核心
    * 不仅是元数据的唯一持久记录
    * 充当定义并发操作顺序的逻辑时间线。（文件和块，以及它们的版本，都是唯一的，由它们创建的逻辑时间来标识）
  * 必须可靠地存储它
    * 它复制到多个远程机器上，只有在本地和远程将相应的日志记录刷新到磁盘之后才响应客户机操作
    * 在刷新之前，Master 批处理多个日志记录，从而减少刷新和复制对总体系统吞吐量的影响。
  * master通过replay操作日志恢复其文件系统状态
    * 必须保持日志较小，每当日志增长超过一定的大小时，Master 检查其状态
    * 旧的检查点和日志文件可以自由删除，但通常还是会保留

#### 2.7 Consistency Model

> 这部分个人理解的不太好，感觉很难用自己的语言进行总结，直接摘抄了[博客](https://spongecaptain.cool/post/paper/googlefilesystem/#35-consistency-model-%E4%B8%80%E8%87%B4%E6%80%A7%E6%A8%A1%E5%9E%8B)中的总结

GFS 有一个宽松的一致性模型，它可以很好地支持高度分布式的应用程序，但是实现起来仍然相对简单和高效。我们现在讨论 GFS 的保证以及它们对应用程序的意义。我们还强调了 GFS 如何维持这些担保，但将细节留给了文件的其他部分。

**Guarantees by GFS**

- 文件名 namespace 命名空间的变化（比如，文件的创建）全权由 Master 节点在内存中进行管理，这个过程通过 namespace lock 确保操作的原子性以及并发正确性，Mater 节点的 operation log 定义了这些操作在系统中的全局顺序；

在数据修改后，文件区域的状态取决于很多个条件，比如修改类型、修改的成功与否、是否存在并发修改，下表总结了文件区域的状态（来自于论文）：

![image-20200719211636393](https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20200719211636393.png)

> 这张表从下到上对并发修改的要求逐渐增高，Failure 表示并发修改失败，Concurrent success 表示并发修改成功，Serial success 则表示串行修改成功，串行要求最高，但是其如同单线程写一样不存在任何并发问题。

为了看懂上表，首先我们应当理解 GFS 系统中对 file region 状态的概念定义：

- consistent：所有 GFS Client 将总是看到完全相同的数据，无论 GFS Client 最终是从哪一个 GFS chunkserver replica 上进行数据读取；

- defined：当一个文件数据修改之后如果 file region 还是保持 consistent 状态，并且所有 client 能够看到全部修改（且已经写入 chunkserver）的内容；

  > 这种说法相当于并发正确地写入成功，并且我们可以看到 defined 的要求高于 consistent，因为后者是前者的基础；

- consistent but undefined：从定义上来看，就是所有 client 能够看到相同的数据，但是并不能及时反映并发修改中的任意修改；

  > 这通常指写冲突发生了，GFS 并不保证多个客户端的并发覆写请求的最终执行顺序，这种顺序是 undefined，这是因为不能确定真正的执行次序而不确定。但是最终所有客户端查询时能读到相同的结果。

- inconsistent：因为处于 inconsistent 状态，因此一定也处于 undeﬁned 状态，造成此状态的操作也被认为是 failed 的。不同的 Client 在能读到的数据不一致，同一个 Client 在不同的时刻读取的文件数据也不一致。

其次，表格将数据的修改分为两种情况：

- Write：修改 File 中的原有数据，具体来说就是在指定文件的偏移地址下写入数据（这就是覆写操作）；

  > GFS 没有为这类覆写操作提供完好的一致性保证：如果多个的 Client 并发地写入同一块文件区域，操作完成后这块区域的数据可能由各次写入的数据碎片所组成，此时的状态最好也就是 consistant but undefined 状态。

- Record Append：即在原有 File 末尾 Append(追加)数据，这种操作被 GFS 系统确保为原子操作，这是 GFS 系统最重要的优化之一。GFS 中的 append 操作并不简单地在原文件的末尾对应的 offset 处开始写入数据（这是通常意义下的 append 操作），而是通过选择一个 offset，这一点在下面会详细说到。最后该被选择的 offset 会返回给 Client，代表此次 record 的起始数据偏移量。由于 GFS 对于 Record Append 采用的是 at least once 的消息通信模型，在绝对确保此次写操作成功的情况下，可能造成在重复写数据。

在一系列成功的修改操作以后，被修改的文件区域的状态是 defined 并包含最后一次修改的写内容。GFS 通过以下两种方式实现这一目标：

- 在所有写操作相关的 replicas 上以同一顺序采用给 chunk 进行修改；

- 使用 chunk version numbers（也就是版本号）去检测 replicas 上的数据是否已经 stale（过时），这种情况可能是由于 chunkserver 出现暂时的宕机(down)；

  > 注意，一旦一个 replicas 被判定为过时，那么 GFS 就不会基于此 replicas 进行任何修改操作，客户机再向 Master 节点请求元数据时，也会自动滤除过时的 replicas。并且 Master 通常会及时地对过时的 replicas 进行 garbage collected（垃圾回收）。

出现的相关问题：

- 缓存未过期时 replica 出现过时的问题：因为在客户机存在缓存 cache 的缘故，在缓存被刷新之前，客户机还是有机会从 stale replica 上读取文件数据。这个时间穿窗口取决于缓存的超时时间设置以及下一次打开同一文件的限制。另一方面，GFS 中的大多数文件都是 append-only，因此 stale replica 上的读仅仅是返回一个 premature end of chunk，也就是说仅仅没有包含最新的追加内容的 chunk，而不是被覆写了的数据（因为无法覆写），这样造成的影响也不会很大。
- 组件故障问题：Master 节点进行通过与所有 chunkserver 进行 regular handshake（定期握手）来检测出现故障的 chunkserver，通过 checksumming（校验和）来检测数据是否损坏。一旦出现问题，GFS 会尽快地从有效的 replicas 上进行数据恢复，除非在 Master 节点检测到故障之前，存储相同内容的 3 块 replica 都出现故障时才会导致不可逆的数据丢失。不过即使是这样，GFS 系统也不会不可用，而是会及实地给客户端回应数据出错的响应，而不是返回出错的数据。

**Implications for Applications** 也就是对使用 GFS 的应用的要求

使用 GFS 的应用可以使用一些简单的技术来达到 GFS 系统所支持的宽松一致性协议，比如：

- 尽量选择 append 追加，而不是 overwrite 覆写，这是因为 GFS 仅仅保证 append 操作的一致性，但是覆写操作没有一致性保证；
- 写入数据时使用额外的校验信息，比如校验和（或者其他 hash 技术）；
- 可以选择加入额外的唯一标识符来去除因为 write at least once 而造成的重复数据（直白一点就是客户端在读取数据时如果发现唯一标识符已经读过了，那么就舍弃这部分数据）；

### 3 System Interactions

设计思想：最小化master的参与

交互目标：data mutations, atomic record append, and snapshot

#### 3.1 Leases and Mutation order

* Mutation
  * 修改chunk内容或是元数据，比如write、append
  * 每个mutation都会作用于chunk的所有副本
* leases：用于维持在副本集上mutation order一致性的机制，具体操作如下
  - Master 节点将一个 chunk lease 发给写操作设计 chunk 的三个 chunkserver 中的任意一个节点，此节点被称为 primary 节点，而其他两个节点被称为 secondaries(从属节点)。Master 在此消息中还会告知被选中的 primary 节点来自客户端的多个写操作请求；
  - chunkserver 收到 lease 以及写操作请求后，其才认为自己有权限进行如下操作：决定多个写操作的执行顺序，此顺序被称为 serial order（串行执行顺序）。
  - 当 primary 决定好顺序后，会将带有执行顺序的 lease 返回给 master 节点，master 节点随后会负责将顺序分发给其他两个 replica。其他两个 replicas 并没有选择权，只能按照 primary 决定的顺序进行执行。
*  Lease 机制减少了 Master 的管理开销，同时也确保了线程安全性，因为执行顺序不再由 Master 去决定，而是由拥有具体 lease 租赁的 chunkserver 节点决定。
* 租赁的默认占用超时时间为 60s，但是如果 Master 又接收到对同一个 chunk 的写操作，那么可以延长当前 primary 节点的剩余租赁时间。
* 即使失联，master也可以在旧lease过期以后重新挑选新的lease

![image-20211213102831116](https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20211213102831116.png)

- Step1：客户端向 Master 节点查询哪一个 Chunk Server 持有要进行写操作的 Chunk 的 Lease；

- Step2：Master 节点回应 primary 节点的标识符（包括地址）以及其他 replicas 节点的地址。客户端接收后将此回应进行缓存，其会在 primary 节点不可达或者其不再持有 lease 时再次向 Master 查询；

- Step3：客户端向所有的 replicas 都推送数据，注意此时客户端可以依靠任意顺序进行推送数据，并没有要求此时必须先给 primary 推送数据。所有的 chunkserver(replicas) 都会将推送来的数据存放在内置的 LRU buffer cacahe，缓存中的数据直到被使用或者超时才会被释放。

  > 这里数据流操作本身不涉及 Master 节点，也达到了给 Master 节点减压的目的。

- Step4：只要 replicas 回复已经接收到了所有数据，那么 Client 就会发送一个 write 指令给 primary 节点，primary 节点为多个写操作计划执行的序号（写操作可能来自于多个 Client），然后将此顺序应用于其本地 I/O 写操作。

- Step5：primary 节点将写操作请求转发给其他两个 replica，它们都将按照 primary 的顺序执行本地的 I/O 写操作；

- Step6：secondaries 从节点返回写成功的响应给 primary 节点；

- Step7：Primary 响应客户端，并返回该过程中发生的错误。注意，这里的错误不仅仅是 Primary 节点的写操作错误，还包括其他两个 replica 节点的写操作错误。如果 primary 自身发生错误，其就不会向其他两个 replica 节点进行转发。另一方面，如果 Client 收到写失败响应，那么其会重新进行写操作尝试，即重新开始 3-7 步。

#### 3.2 Data Flow

GFS解耦了数据流和控制流，目的是充分利用每台机器的网络带宽，避免网络瓶颈和高延迟链接

* 数据流被线性传输：
  * Client 负责给 primary 传输写入的数据，而 primary 负责给下一个 replica 传输数据，而下一个 replica 又负责给下下一个 replica 传输数据
  * 每台机器会向网络拓扑中最近且未接受的机器发起传输
* Pipeline
  * 当一个 chunkserver 接收到一些数据后（不是一个字节就会触发，而是要有一定阈值）就会立即转发给下一个节点，而不是等到此次写操作的所有数据接收完毕，才开始向下一个节点转发。
  * GFS 系统使用的是全双工网络，即发送数据时不会降低数据接收速率。

#### 3.3 Atomic Record Appends

客户端仅仅负责指定要写的数据，GFS 以 at least once 的原子操作进行写，写操作的相关数据一定是作为连续的字节序列存放在 GFS 选择的偏移量处。

GFS record append 操作的内部执行逻辑如下：

- Client 确定 file name 以及要写入的 byte data（形式上可以选择一个 buffer 来存储要写入的字节数据）；
- Client 向 Master 发出期望进行 record 操作的请求，并附带上 file name，但是不用携带字节数据；
- Master 接收到请求之后得知是一个 append record 请求，并且得到 file name。Master 节点通过其内存中的 metadata 得到当前 file 分块存储的最后一个 chunk 的 chunk handle 以及 chunk 所在的所有 chunkserver；
- Master 之后将这些信息转发给 Client；
- 后面的操作就类似于 3.1 小节中的过程。

Record append 操作还涉及 primary 的选择步骤：

- Master 节点在接受到修改请求时，会找此 file 文件最后一个 chunk 的 up-to-date 版本（最新版本），最新版本号应当等于 Master 节点的版本号；

  > 什么叫最新版本。chunk 使用一个 chunk version 进行版本管理（分布式环境下普遍使用版本号进行管理，比如 Lamport 逻辑时钟）。一个修改涉及 3 个 chunk 的修改，如果某一个 chunk 因为网络原因没能够修改成功，那么其 chunk version 就会落后于其他两个 chunk，此 chunk 会被认为是过时的。

- 递增当前 chunk 的 chunk version 并通知 Primary，得到确切回复后再本地持久化（https://tanxinyu.work/gfs-thesis/）

- Primary 然后开始选择 file 最后一个文件的 chunk 的末尾 offset 开始写入数据，写入后将此消息转发给其他 chunkserver，它们也对相同的 chunk 在 offset 处写入数据；

这里有几个注意要点：

**如果向 file 追加的数据超过了 chunk 剩余容量怎么办？**

- 首先，这是一个经常发生的问题，因为 record append 操作实际上能一次添加的数据大小是被限制的，大小为 chunksize（64 MB）的 1/4，因此在大多数常见下，向 chunk append 数据并不会超出 64 MB 大小的限制；
- 其次，如果真的发生了这个问题，那么 Primary 节点还是会向该 chunk append 数据，直到达到 64MB 大小上限，然后通知其他两个 replicas 执行相同的操作。最后响应客户端，告知客户端创建新 chunk 再继续填充，因为数据实际上没有完全消耗掉；

**失败处理？**

> 特别值得一提的是，GFS保证追加操作至少被执行一次（at least once），这意味着追加操作可能被执行多次。当追加操作失败时，为了保证偏移量，GFS会在对应的位置填充重复的数据，然后重试追加。也就是说，GFS不保证在每个副本中的数据完全一致，而仅仅保证数据被写入了。

* 如果是write失败，那么客户端可以重试，直到write成功，达到一致的状态。但是如果在 重试达到成功以前出现宕机，那么就变成了永久的不一致了。
* Record Append在写入失败后，也会重试，但是与write的重试不同，不是在原有的offset 上重试，而是接在失败的记录后面重试，这样Record Append留下的不一致是永久的不一 致，并且还会有重复问题，如果一条记录在一部分副本上成功，在另外一部分副本上失败，那么这次Record Append就会报告给客户端失败，并且让客户端重试，如果重试后成功，那么在某些副本上，这条记录就会成功的写入2次。

![image-20211213111917356](https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20211213111917356.png)

#### 3.4 Snapshot

* 对文件和目录树做copy，并尽量不影响正在进行的mutations操作
* 采用 **copy-on-write** 技术
  * Master 节点收到一个 snapshot 请求，它首先会 revoke（撤销） 对拷贝涉及的 chunk 对应的 lease（租赁），这用于确保后续写操作必须经过 Master 的重新进行交互，进行查找当前租赁的实际持有者，以便于 Master 有机会先创建 chunk 的副本。
  * 当 lease 撤回或者过期后，Master 首先会将操作日志记录到磁盘（因为要修改metadata），然后通过复制源文件以及目录树的 metadata 来将日志记录应用到内存中的状态。
  * 当有关客户端请求 Master 对这些 chunk 进行写操作时，Master 通过这些 chunk 上的引用计数大于 1，于是 Master 就会为这些 chunk 创建相关的 handler，然后通知拥有这些 chunk 的 chunkserver 创建数据相同的 chunk（这种方式不再 Master 上进行复制，目的是节约 Master 带宽与内存）。
  * 最后客户端新的写请求将直接作用于这些新创建的 chunk 上，同时也会被颁发新的 lease；

### 4 Master Operation

- 所有 namespace 的管理工作；
- 管理整个系统中的所有 chunk replicas：
  - 做出将 chunk 实际存储在哪里的决定；
  - 创建新的 chunk 和 replica；
  - 协调系统的各种操作（比如读、写、快照等），用于保证 chunk 正确且有效地进行备份；
  - 管理 chunkserver 之间的负载均衡；
  - 回收没有被收用的存储空间；

#### 4.1  Namespace Management and Locking

* GFS 逻辑上将其 namesapace 当做一个查询表，用于将 full pathname（要么是一个 absolute file name，要么是一个 absolute directory name） 映射为 metadata。如果使用 prefix compression（前缀压缩）
* 锁逻辑
  * 最底层的文件/目录一定是获得写锁
  * 除了最底层的文件/目录，其他所有父目录、祖父目录仅仅需要获得读锁，读锁是一种共享锁。
  * 例如，要写`/home/user/foo`，就要获得读锁`/home`、`/home/user`和写锁`/home/user/foo`

#### 4.2 Replica Placement

Chunk replicas分配的目标：

1. 最大化数据可靠性和可获得性
2. 最大化网络带宽利用率

#### 4.3 Creation, Re-replication, Rebalancing

chunk副本可能因为三个原因被创建：

* chunk creation
  * mater选择在哪里创建副本，有三个原则
    * 希望放置在磁盘利用率较低的chunkserver上
    * 限制每个chunkserver“最近创建的chunk”数，虽然创建chunk的代价不高，但通常是即将出现大量写操作的前兆
    * 正如4.2的讨论，希望将副本创建在不同的机架上
* re-replication
  * 当可用副本数<设定值时，可能因为：
    * chunkserver失联
    * 报告磁盘损坏
    * 设定值被提高
  * master进行re- replication时候依照以下优先级：
    * 根据距离 replication goal 的配置的距离来确定优先级。比如默认三个 replicas，有一组 replicas 中有两个出现了不可用，而另一组仅仅只有一个出现了不可用，因此前者比后有优先级高；
    * 最近活动的文件（被读、被写）比最近删除的文件的 chunk 有更高的优先级；
    * 如果 chunk 的读写可能阻塞客户端，那么该 chunk 将有较高的优先级，这能够减少 chunk 故障时对使用 GFS 的应用程序的影响；
  * master会挑选优先级最高的chunk进行clone操作，通过直接指示chunkserver进行chunk的复制来实现，新chunk的规则同chunk creation
* rebalancing
  * 定期进行
  * 检查当前 replica 的分布，然后会将相关 replicas 移动到更好的磁盘位置。
  * 将一个新加入的 chunkserver 自动地**逐渐**填充，而不是立即在其刚加入时用大量的写操作来填充它

#### 4.4 Garbage Collection

当文件被删除时，GFS 并不会立即回收文件的物理磁盘存储空间，而是提供了一个 regular(定期的)的垃圾收集机制，用于回收 file 和 chunk 级别的物理磁盘空间

**机制说明**

* File GC
  * 当文件被删除时，master log 记录被删除的文件，然后将文件重命名为 hadden name（隐藏名），文件的重命名工作仅仅在 Master 的 namespace 中进行，此名字包含接收到删除指令的时间戳
  * 定期对 namespcae 的扫描过程中，其会移除所有距删除时间戳 3 天以上的 hidden files，3 天超时时间可以通过配置修改。在 3 天之内，hidden files 还是可以通过 hidden name 进行读取，不过原名是不行了。并且可以将文件恢复为正常名而撤销删除操作。
  * 当 hidden file 从 namespace 中移除后，该文件在 Master 节点中的所有 metadata 都被移除了，然后所有的 chunkserver 会删除在磁盘上的相关文件。
* Chunk GC
  *  master会定位孤立chunk（即不被任何file持有），并且将这些chunk的metadata删除
  * 在心跳信息中，每个chunkserver会带上持有chunk的一个子集，每次master回复的时候都会进行检查，如果存在某个chunk的metadata已经被删除，就会通知chunkserver将其删除

#### 4.5 Stale Replica Detection

当 chunkserver 故障了或者因为宕机没能够正确地实施写操作，那么 Chunk replicas 的状态就变为 stale

Master 节点为每一个 chunk 维护一个 chunk verison nembe 来辨别哪些 chunk 是最新的，哪些 chunk 是过期的

* master lease时候，会增加版本号，并通知给replicas使其更新。这个操作会优先于写操作执行前发生，如果在lease的时候有chunkserver失联，那么其 chunk version 就不会增加
* 当此 Chunkserver 重启后的心跳消息中就会包含此 chunk version 信息，Master 就能借此发现 ChunkServer 拥有 stale 的 replica
* master会在GC的时候删除stale replicas。在此之前，它在答复客户机请求获取信息时，实际上认为过时的复制根本不存在。

### 5 Fault tolerance and Diagnosis

#### 5.1 高可用性

* Fast Recovery
  * Master 以及 chunkserver 节点不管出于何种原因故障，都能在几秒内恢复故障前的状态。GFS 并不对主机的正常关闭和异常关闭进行区别看待
* Chunk Replication
* Master Replication
  * 日志系统

#### 5.2 数据完整性

每个chunkserver使用校验和来检验数据是否损坏

* 磁盘损坏是经常发生的，我们可以从其他副本中恢复数据，但是通过跨服务器比较副本来检测损坏是不实际的，而且不同的副本数据不一致可能是合法的（at lease once）
* 每个chunk每一个 chunk被分为 64KB 大小的小块，这意味着默认情况下一个 chunk 对应 1k 个 block。每一个小块都有对应的 32 bit校验和，校验和数据和用户数据分开存储，和其他元数据一样，一起保存在内存中，并最终通过日志系统持久化

读数据：

1. chunkserver 对读操作涉及的所有 block 块进行校验。如果 chunkserver 发现校验和和数据不匹配，那么就会向请求者返回一个错误，同时还向 Master 节点报告错误。
2. 请求者接收到此响应后，会从其他 replica 上读取数据。
3. Master 接收到此响应后，将从另一个 replica 上克隆数据块，并指示 chunkserver 删除它的 replica。

append：

* 只需要增加最后部分 block 的 checksum 即可。即使最后一个部分校验和块已经损坏，而我们现在没有检测到它，新的校验和值也不会与存储的数据匹配
* 当下一次读取块时，还是会被检测到损坏。但是覆写操作的校验和可能会失效，因为 GFS 并没有进行特别的优化。

write：

* 没有额外优化

空闲时：

* chunkservers 在空闲期间可以扫描和验证非活动 chunk 的内容。通过这种机制能够对很少被读取的 chunk 也进行数据是否损坏的检测，一旦检测到损坏，主服务器可以创建一个新的未损坏的副本并删除损坏的副本
* 这样可以防止 Master 错误认为对于一个不被经常读取的 chunk 有着符合配置数量要求的 replicas



## Lecture

### Master Data

2 tables stored in RAM：

log, checkpoint in disk

![image-20211215143225404](https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20211215143225404.png)

> NV: 易失，需要持久化到log
>
> V：不需要持久化 

version number 需要持久化，因为有时候master重启的时候持有最新chunk的server还在宕机，如果这时候用懒加载

### 脑裂

SPLIT BRAIN

3.2 中，我们提到“即使失联，master也可以在旧lease过期以后重新挑选新的lease”

为什么不立即指定一个新的primary？

* 失联可能只是因为master和primary的网络出现异常（“网络分区”异常），而clint和primary之间的连接正常
* 此时如果选择了一个新的primary，由于客户端缓存的存在，会继续和旧primary通信，这时候有两个primary，会出现冲突，称为“脑裂”

### 强一致性的讨论

讨论：如何将GFS升级为强一致性的系统，需要考虑哪些？

![image-20211215174744075](https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20211215174744075.png)

1. 让primary检查重复的请求，比如重试append B的这条，以保证B不会被重复写入 
2. primary让secondary做某事，secondary必须执行，而不只是返回错误；当secondary发生错误的时候，要退出系统。 

也就是说，将写操作分为多个阶段，两阶段提交（Two-phase commit），在未完成前不应该让client感知

* 第一个阶段，Primary向Secondary发请求，要求其执行某个操作，并等待Secondary回复说能否完成该操作，这时Secondary并不实际执行操作。
* 第二个阶段，如果所有Secondary都回复说可以执行该操作，这时Primary才会说，好的，所有Secondary执行刚刚你们回复可以执行的那个操作。