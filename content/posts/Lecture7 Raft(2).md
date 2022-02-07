+++

title = "MIT6.824 Lecture7 Raft-2"

date = "2022-02-07"

tags = [
    "MIT6.824",
    "distributed system",
]

+++

# Lecture7 Raft(2)

> 讲义：http://nil.csail.mit.edu/6.824/2020/notes/l-raft2.txt
>
> 视频：https://www.bilibili.com/video/BV1qk4y197bB?p=7
>
> 参考笔记：https://mit-public-courses-cn-translatio.gitbook.io/mit6-824/lecture-07-raft2
>
> 对应raft论文第七章至结束

## 选举约束

Raft的核心思想：在选举时保证成为leader的节点一定有**最新**的committed日志（而不是最长的）

1. 候选人最后一条Log条目的任期号**大于**本地最后一条Log条目的任期号；

2. 或者，候选人最后一条Log条目的任期号**等于**本地最后一条Log条目的任期号，且候选人的Log记录长度**大于等于**本地Log记录的长度

## 快速备份

如果Log有冲突，Leader每次会回退一条Log条目。但如果冲突较多，开销会很大，这里Morris教授总结了几个快速备份的方法。

![image-20220207140153794](https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220207140153794.png)

> 回顾：AppendEntriesRPC的请求体中会带有`previousTerm`和`previousIndex`

可以让Follower在回复Leader的AppendEntries消息中，携带3个额外的信息，来加速日志的恢复。这里的回复是指，Follower因为Log信息不匹配，拒绝了Leader的AppendEntries之后的回复。这里的三个信息是指：

- XTerm：这个是Follower中与Leader冲突的Log对应的任期号；如果Follower在对应位置没有Log，那么这里会返回 -1。
- XIndex：这个是Follower中，对应任期号为XTerm的第一条Log条目的槽位号。
- XLen：如果Follower在对应位置没有Log，那么XTerm会返回-1，XLen表示Follower log的长度。

用快速备份来解决上述三个case

* case1：Follower（S1）会返回XTerm=5，XIndex=2。Leader（S2）发现自己没有任期5的日志，它会将自己本地记录的，S1的nextIndex设置到XIndex。

* case2：Follower（S1）会返回XTerm=4，XIndex=1。Leader（S2）发现自己其实有任期4的日志，它会将自己本地记录的S1的nextIndex设置到本地在XTerm位置的最后一条Log条目后面，也就是槽位2。

* case3：Follower（S1）会返回XTerm=-1，XLen=1。Leader（S2）会将自己本地记录的S1的nextIndex设置到本地在XLen+1的位置，也就是槽位2。

## 持久化

Raft server 中有些数据被标记为持久化的（Persistent），有些信息被标记为非持久化的（Volatile）

Persistent

* log
* currentTerm
* votedFor：为了保证在每个term只投票给一位candidate

## 日志压缩和快照

InstallSnapshotRPC





## 线性一致性Linearizability

什么是正确的？

参考：https://tanxinyu.work/consistency-and-consensus/

* 例一：

  <img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220207172731854.png" alt="image-20220207172731854" style="zoom:50%;" />

  满足线性一致性

* 例二：

  <img src="https://cyzblog.oss-cn-beijing.aliyuncs.com/macimg/image-20220207173049085.png" alt="image-20220207173049085" style="zoom:50%;" />

  存在环，不满足线性一致性