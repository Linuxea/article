# A Deep Dive Into Go Concurrency

> 从并发角度最为健壮的编程语言


Golang 为人所知是其对并发的头等支持，或者说一种程序在同一时刻处理许多事情的能力。

在计算机从更快地运行单一的代码流演变为同时运行更多代码流的过程中，并发正在成为编程中越来越重要的一部分。


程序员能够通过设计代码相互彼此独立并发执行来让 ta 们的程序运行更快。

Go 中三个特性：`goroutine`, `channel` 与 `select` 的结合使用让并发更加简单。

Goroutine 解决了程序并发运行的问题，channel 解决了并发程序代码之间的通信问题。

其中，Goroutine 毫无疑问是 Go 的最大特性！
它们非常轻量级，不像操作系统线程，而是通过将成千上万的协程复用到一个操作系统线程（Go 为此有运行调度器）伴随着最小的上下文切换开销好处。

简单来说，即协程是线程的轻量级与廉价的抽象。

## Go Runtime Scheduler

可以说，Go 运行时调度器的工作是分发可运行的协程运行在一个或多个处理器的一个或多个线程上。

- G = Goroutine
- M = OS Thread
- P = Processor

处理器处理多个线程。线程处理多个协程。

处理器是硬件相关的，处理器的数量取决于 CPU 核心数量。

当新的协程被创建出来，或者一个已经存在的协程变为可运行状态，它会被推到当前处理器的一个可运行协程列表。


## What's a Goroutine?

Goroutine 是可与其他函数一起运行的函数。

Goroutine 可以被认为是操作系统线程之上的轻量级线程。相比操作系统线程，创建 Goroutine 的代价是微小的。

因此对于一个 Go 应用来讲，成千上万的协程并发运行是常见且普通的。

许多的协程被复用到很少数量的操作系统线程中。

程序中存在仅仅一个线程与成千上万的协程的是有可能的。


一个协程会有三种状态：running(运行中)，runnable(可运行), not runnable(不可运行)。


## Goroutines vs. Threads

创建一个协程并不需要太多的内存-只需 2kb 的栈空间。当需要的时候通过分配与释放堆存储。

作为对比，线程启动的时候伴随着较大的空间，以及一个称为保护页面的内存区域，它充当一个线程的内存和另一个线程之间的保护。

协程在运行时容易创建与销毁，不过线程拥有更大的启动与停止代价：它需要从操作系统处请求与归还资源。

运行时少数的几个线程会被分配给所有协程复用。

在系统的任何时刻，每个线程都会只执行一个协程。

如果协程被阻塞了（函数调用，系统调用，网络调用，等待），它会被替代为另外一个协程在此线程上执行。

总而言之，Go 使用了协程与线程，两者在函数的并发执行的组合是必不可少的。但是协程的使用让 Go 变成比一开始看起来成为了一门更好的编程语言。



## Goroutines Queues

Go 从两个层面来管理协程，本地队列与全局队列。本地队列依附在每个处理器，而全局队列是通用且唯一的。

每个本地队列拥有的最大容量为 256，并且之后任何新来的协程都会被的念头到全局队列。

协程会在本地队列满时进入全局队列，并且当 Go 向调度器注入协程列表时也会被推到全局队列，比如来自网络轮询器或者发生 GC 时睡眠的协程。


## Stealing Work

当一个处理器没有任何工作时，以下规则会被按顺序应用直到任一满足：

- 从本地队列拉取
- 从全局队列拉取
- 从网络轮询器拉取
- 从其他处理中窃取

因为处理器可以在任务用完时从全局队列提取工作，第一个可用的处理器会运行这些协程。这种行为解释了为什么一个协程可以运行在不同的 P 上，并且展示了在资源空闲时 Go 是怎样通过让其他协程运行来优化系统的。

## Run and Debug

在这里的代码片断中，我们创建了 20 个协程函数。

每个函数会睡眠一秒然后计数到 1e10(10,000,000,000).

我们通过设置环境变量 `GODEBUG=schedtrace=1000` 来调试 Go 调度器。


### Code

```golang
package main

import (
    "sync"
    "time"
)

var wg sync.WaitGroup

func main() {
    for i := 0; i < 20; i++ {
        wg.Add(1) // increases WaitGroup
        go work() // calls a function as goroutine
    }

    wg.Wait() // waits until WaitGroup is <= 0
}

func work() {
    time.Sleep(time.Second)

    var counter int

    for i := 0; i < 1e10; i++ {
        counter++
    }

    wg.Done()
}
```


### Results


结果展示了协程全局队列 runqueue 以及大括号中本地队列的大小。

- gomaxprocs: 处理器配置最大数量
- idleprocs: 没有处于使用状态的处理器
- threads: 使用中的线程
- idlethreads: 没有使用的线程
- runqueue: 全局队列中的协程
- `[1 0 0 0 0 0 0 0 0 0 0 0]`: 各个处理器中的本地协程队列

```log
➜  gotour GODEBUG=schedtrace=1000 go run main.go
SCHED 0ms: gomaxprocs=12 idleprocs=11 threads=3 spinningthreads=0 idlethreads=0 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
# command-line-arguments
SCHED 0ms: gomaxprocs=12 idleprocs=9 threads=5 spinningthreads=1 idlethreads=1 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
# command-line-arguments
SCHED 0ms: gomaxprocs=12 idleprocs=9 threads=5 spinningthreads=1 idlethreads=0 runqueue=0 [1 0 0 0 0 0 0 0 0 0 0 0]
SCHED 0ms: gomaxprocs=12 idleprocs=9 threads=5 spinningthreads=1 idlethreads=1 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 1000ms: gomaxprocs=12 idleprocs=12 threads=20 spinningthreads=0 idlethreads=13 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 1003ms: gomaxprocs=12 idleprocs=1 threads=12 spinningthreads=1 idlethreads=0 runqueue=0 [4 4 0 0 0 1 0 1 0 0 0 0]
SCHED 2001ms: gomaxprocs=12 idleprocs=12 threads=20 spinningthreads=0 idlethreads=13 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 2010ms: gomaxprocs=12 idleprocs=0 threads=13 spinningthreads=0 idlethreads=0 runqueue=8 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 3002ms: gomaxprocs=12 idleprocs=12 threads=20 spinningthreads=0 idlethreads=13 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 3017ms: gomaxprocs=12 idleprocs=0 threads=13 spinningthreads=0 idlethreads=0 runqueue=7 [0 0 0 0 0 0 0 0 1 0 0 0]
SCHED 4008ms: gomaxprocs=12 idleprocs=12 threads=20 spinningthreads=0 idlethreads=13 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 4023ms: gomaxprocs=12 idleprocs=0 threads=13 spinningthreads=0 idlethreads=0 runqueue=8 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 5015ms: gomaxprocs=12 idleprocs=12 threads=20 spinningthreads=0 idlethreads=13 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 5031ms: gomaxprocs=12 idleprocs=0 threads=13 spinningthreads=0 idlethreads=0 runqueue=8 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 6023ms: gomaxprocs=12 idleprocs=12 threads=20 spinningthreads=0 idlethreads=13 runqueue=0 [0 0 0 0 0 0 0 0 0 0 0 0]
SCHED 6037ms: gomaxprocs=12 idleprocs=0 threads=13 spinningthreads=0 idlethreads=0 runqueue=8 [0 0 0 0 0 0 0 0 0 0 0 0]
```


## 参考

- [1] [A Deep Dive Into Go Concurrency](https://betterprogramming.pub/deep-dive-into-concurrency-of-go-93002344d37b)
- [2] [Go 调度器跟踪](https://colobu.com/2016/04/19/Scheduler-Tracing-In-Go/)
- [3] [Go: Work-Stealing in Go Scheduler](https://medium.com/a-journey-with-go/go-work-stealing-in-go-scheduler-d439231be64d)