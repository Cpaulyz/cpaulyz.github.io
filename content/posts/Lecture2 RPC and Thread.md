+++

title = "MIT6.824 Lecture2  Go, RPC and Thread"

date = "2021-12-09"

tags = [
    "MIT6.824",
    "distributed system",
]

+++

# Lecture2 RPC and Thread

讲义：http://nil.csail.mit.edu/6.824/2020/notes/l-rpc.txt

视频：https://www.bilibili.com/video/BV1R7411t71W?p=2

## Go

### why go？

* good support for threads
* convenient RPC
* type safe：memory safe
* garbage-collected (no use after freeing problems)
* threads + GC is particularly attractive!
  * 相比cpp，简单很多，不用判断哪个线程最后使用共享变量并负责释放
* relatively simple
* After the tutorial, use https://golang.org/doc/effective_go.html

## Thread

> Go calls them goroutines; everyone else calls them threads

### Why care？

* 实现并发的重要工具
* 并发是本课程的重点

### Why thread？

* They express concurrency, which you need in distributed systems
* I/O concurrency
  * 多个thread，向server并行地发起RPC，等待回应
  * Client sends requests to many servers in parallel and waits for replies.
  * Server processes multiple client requests; each request may block.
  * While waiting for the disk to read data for client X, process a request from client Y.
* Multicore performance
  * Execute code in parallel on several cores.
* Convenience
  * 想在后台做一些事情，例如每过1s检查worker是否存活

### 替代品：event-driven

Is there an alternative to threads?

Yes: write code that explicitly interleaves activities, in a single thread.

Usually called "event-driven." aka asynchronous programming

Keep a table of state about each activity, e.g. each client request.

One "event" loop that:

* checks for new input for each activity (e.g. arrival of reply from server),
* does the next step for each activity,
* updates state.

Event-driven gets you I/O concurrency and eliminates thread costs (which can be substantial),but doesn't get multi-core speedup, and is painful to program.

### challenge

* share data 共享内存
  * 不同的写成可以读写同一个东西，可能存在一些临界区
    * 例如，两个线程都在执行 n=n+1，机器码可能如
    * LD x,r1
    * ADD 1,r1
    * STORE r1,x
    * 结果可能会不正确
  * 称为竞争（“Race”）
  * 解决办法：加锁（比如go里的sync.Mutex）、或是直接避免使用共享变量
* coordination
  * 例如，生产消费者问题
  * 特意地互相等待
  * use **Go channels** or **sync.Cond** or **WaitGroup**
* deadlock