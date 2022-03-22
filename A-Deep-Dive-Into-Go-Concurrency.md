# A Deep Dive Into Go Concurrency

> The most robust programming language in terms of concurrency
> 从并发角度来看最为健壮的编程语言


Go is known for its first-class support for concurrency, or the ability for a program to deal with multiple things at once. Running code concurrently is becoming a more critical part of programming as computers move from running a single code stream faster to running more streams simultaneously.

Golang 为人所知是其对并发的头等支持，或者说一种程序在同一时刻处理许多事情的能力。

在计算机从更快地运行单一的代码流演变为同时运行更多代码流的过程中，并发正在成为编程中越来越重要的一部分。


A programmer can make their program run faster by designing it to run concurrently so that each part of the program can run independently of the others. Three features in Go, goroutines, channels, and selects, make concurrency easier when combined together.

程序员能够通过设计代码相互彼此独立并发执行来让 ta 们的程序运行更快。

Go 中三个特性：goroutine, channel 与 select 的结合使用让并发更加简单。


Goroutines solve the problem of running concurrent code in a program, and channels solve the problem of communicating safely between concurrently running code.

Goroutine 解决了程序并发运行的问题，channel 解决了并发程序代码之间的通信问题。

Goroutines are without a doubt one of Go's best features! They are very lightweight, not like OS threads, but rather hundreds of Goroutines can be multiplexed onto an OS Thread (Go has its runtime scheduler for this) with a minimal overhead of context switching! In simple terms, goroutines are a lightweight and a cheap abstraction over threads.

其中，Goroutine 毫无疑问是 Go 的最大特性！
它们非常轻量级，不像操作系统线程，而是通过将成千上万的协程复用到一个操作系统线程（Go 为此有运行调度器）伴随着最小的上下文切换开销好处。

简单来说，即协程是线程的轻量级与廉价的抽象。

## Go Runtime Scheduler

So to say, its job is to distribute runnable goroutines (G) over multiple worker OS threads (M) that run on one or more processors (P). Processors are handling multiple threads. Threads are handling multiple goroutines. Processors are hardware depended; the number of processors is set on the number of your CPU cores.

可以说，Go 运行时调度器的工作是分发可运行的协程到运行在一个或多个处理器的一个或多个线程上。

- G = Goroutine
- M = OS Thread
- P = Processor

处理器处理多个线程。线程处理多个协程。

处理器是硬件相关的，处理器的数量取决于 CPU 核心数量。



When a new goroutine is created, or an existing goroutine becomes runnable, it is pushed onto a list of runnable goroutines of the current processor. When the processor finishes executing a goroutine, it first tries to pop a goroutine from its list of runnable goroutines. If the list is empty, the processor chooses a random processor and tries to steal half of the runnable goroutines.

当新的协程被创建出来，或者一个已经存在的协程变为可运行状态，它会被推到当前处理器的一个可运行协程列表。

当处理器完成对一个协程的执行时，处理器会首先从自己的可运行协程列表中取出一个协程。如果列表是空的，处理器会选择一个随机的处理器并尝试窃取其一半可运行协程。


## What's a Goroutine?

Goroutines are functions that run concurrently with other functions. Goroutines can be considered lightweight threads on top of an OS thread. The cost of creating a Goroutine is tiny when compared to a thread. Hence it's common for Go applications to have thousands of Goroutines running concurrently.

Goroutine 是可与其他函数一起运行的函数。

Goroutine 可以被认为是操作系统线程之上的轻量级线程。相比操作系统线程，创建 Goroutine 的代价是微小的。

因此对于一个 Go 应用来讲，成千上万的协程并发运行是常见且普通的。

Goroutines are multiplexed to a fewer number of OS threads. There might be only one thread in a program with thousands of goroutines. If any Goroutine in that thread blocks says waiting for user input, then another OS thread is created, or a parked (idled) thread is pulled, and the remaining Goroutines are moved to the created or unparked OS thread. All these are taken care of by Go's runtime scheduler. A goroutine has three states: running, runnable, and not runnable.

许多的协程被复用到很少数量的操作系统线程中。

程序中存在仅仅一个线程与成千上万的协程的是有可能的。

如果一个协程在线程中因为等待诸如用户输入时，另外一个操作系统线程会被创建出来，或者一个因空闲而挂起的线程会被拉过来，阻塞的线程中其余协程会被移动到新的或者空闲的操作系统。所有的这一切被 Go 的运行时调度器悉心照顾。

一个协程会有三种状态：running(运行中)，runnable(可运行), not runnable(不可运行)。


## Goroutines vs. Threads

Creating a goroutine does not require much memory, only 2kB of stack space. They grow by allocating and freeing heap storage as required. In comparison, threads start at a much larger space, along with a region of memory called a guard page that acts as a guard between one thread's memory and another.

创建一个协程并不需要太多的内存-只需 2kb 的栈空间。当需要的时候通过分配与释放堆存储。作为对比，线程启动的时候伴随着较大的空间，以及一个称为保护页面的内存区域，它充当一个线程的内存和另一个线程之间的保护。

Goroutines are easily created and destroyed at runtime, but threads have a large setup and teardown costs; it has to request resources from the OS and return it once it's done.

协程在运行时容易创建与销毁，不过线程拥有更大的启动与停止代价：它需要从操作系统处请求与归还资源。

The runtime is allocated a few threads on which all the goroutines are multiplexed. At any point in time, each thread will be executing one goroutine. If that goroutine is blocked (function call, syscall, network call, etc.), it will be swapped out for another goroutine that will execute on that thread instead.


运行时少数的几个线程会被分配给所有协程复用。

在系统的任何时刻，每个线程都会只执行一个协程。

如果协程被阻塞了（函数调用，系统调用，网络调用，等待），它会被替代为另外一个协程在此线程上执行。

In summary, Go is using Goroutines and Threads, and both are essential in their combination of executing functions concurrently. But Go is using Goroutines makes Go a much greater programming language than it might look at first.

总而言之，Go 使用了协程与线程，两者在函数的并发执行的组合是必不可少的。但是协程的使用让 Go 变成比一开始看起来成为了一门更好的编程语言。



## Goroutines Queues

Go manages goroutines at two levels, local queues and global queues. Local queues are attached to each processor, while the global queue is common.

Go 从两个层面来管理协程，本地队列与全局队列。本地队列依附在每个处理器，而全局队列是通用的。

Goroutines do not go in the global queue only when the local queue is full, and they are also pushed in it when Go injects a list of goroutines to the scheduler, e.g., from the network poller or goroutines asleep during the garbage collection.

协程会在本地队列满时进入全局队列，并且当 Go 向调度器注入协程列表时也会被推到全局队列，比如来自网络轮询器或者发生 GC 时睡眠的协程。



## Stealing Work

When a processor does not have any Goroutines, it applies the following rules in this order:
pull work from the own local queue
pull work from network poller
steal work from the other processor's local queue
pull work from the global queue

当一个处理器没有拥有协程时，以下规则会被按顺序应用：
- 从本地队列拉取
- 从网络轮询器拉取
- 从其他处理中拉取
- 从全局队列拉取

Since a processor can pull work from the global queue when it runs out of tasks, the first available P will run the goroutine. This behavior explains why a goroutine runs on different P and shows how Go optimizes the system by letting other goroutines run when a resource is free.

因为处理器可以在任务用完时从全局队列提取工作，第一个可用的处理器会运行这些协程。这种行为解释了为什么一个协程可以运行在不同的 P 上，并且展示了在资源空闲时 Go 是怎样通过让其他协程运行来优化系统的。

In this diagram, you can see that P1 ran out of goroutines. So the Go's runtime scheduler will take goroutines from other processors. If every other processor run queue is empty, it checks for completed IO requests (syscalls, network requests) from the netpoller. If this netpoller is empty, the processor will try to get goroutines from the global run queue.

在上面的这副图中，你可以看到 P1 用完了协程。所以 Go 运行时调度器会从其他处理器获取协程。

如果其他每个协程运行队列都是空的，它会从网络轮询器检查完成的 IO 请求（系统调用，网络请求）。

如果网络轮询器是空的，会尝试从全局队列中获取协程。



## Run and Debug

In this code snippet, we create 20 goroutine functions. Each will sleep for a second and then counting to 1e10 (10,000,000,000). Let's debug the Go Scheduler by setting the env to GODEBUG=schedtrace=1000.

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

The results show the number of goroutines in the global queue with runqueue and the local queues (respectively P0 and P1) in the bracket [5 8 3 0]. As we can see with the grow attribute, when the local queue reaches 256 awaiting goroutines, the next ones will stack in the global queue.


结果展示了协程全局队列 runqueue 以及大括号中本地队列的大小。

正如我们所看到随着属性的增长，当本地队列等待协程数量达到 256 个时，下一个协程将会被放到全局队列中。

- gomaxprocs: 处理器配置
- idleprocs: 没有处于使用状态的处理器
- threads: 使用中的线程
- idlethreads: 没有使用的线程
- runqueue: 全局队列中的协程
- [1 0 0 0 0 0 0 0 0 0 0 0]: 各个处理器中的本地协程队列

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

