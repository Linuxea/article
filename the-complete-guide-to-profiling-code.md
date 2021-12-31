Go 编程语言常常用于性能表现关键的应用.

基于猜想的方式来优化你的代码显然并不是最佳的实践.

你需要深入洞察你的代码性能与瓶颈,来达到更有效率地优化.



## What is a profile ?

Profiler 是一个动态性能分析工具,可以从多个维度提供关键的执行洞察来解决性能问题,定位内存泄露,线程竞争等等.



## What kinds of profiles can I get?

Go 内置了多种 profiles:

- Goroutine: 当前所有协程的堆栈跟踪
- CPU: 运行时返回的 CPU 堆栈跟踪
- Heap: 存活对象的内存分配示例
- Allocation: 所有包括过去的内存分配示例
- Thread: 导致创建新系统线程的堆栈追踪
- Block: 导致在同步原语上阻塞的堆栈追踪
- Mutex: 竞争互斥持有体的堆栈追踪



## How to get a profile?

### Using 'go test' to generate profile

标准的 testing 包内置了对 profiling 的支持.

```
go test -cpuprofile cpu.prof -memprofile mem.prof -bench=.
```

上述例子,会运行所有的测试的同时将 CPU 与 内存的信息写入到 cpu.prof 与 mem.prof 文件中.




### Download live profile using HTTP


如果需要对一个长期运行的服务进行 profile, net/http/pprof 是你的最终方案.
在代码中引入包

```golang
import _ "net/http/pprof"
```

net/http/pprof 包中的初始化代码块是你需要导入这个包的理由:
```golang
func init() {
	http.HandleFunc("/debug/pprof/", Index)
	http.HandleFunc("/debug/pprof/cmdline", Cmdline)
	http.HandleFunc("/debug/pprof/profile", Profile)
	http.HandleFunc("/debug/pprof/symbol", Symbol)
	http.HandleFunc("/debug/pprof/trace", Trace)
}
```


初始化代码块将几个 handler 以给定的 URL 注册到了 DefaultServeMux.
这些 handler 将 profile 写入到 http.ResponseWriter 中以便可以下载.

而你需要做的事就是启动一个 handler 为 nil(会使用默认的 DefaultServeMux) 的 HTTP 服务
```golang
func init() {
	go func() {
		http.ListenAndServe(":1234", nil)
	}()
}
```

> Note: 如果你已经在存在的代码存在并启动 DefaultServerMux 的 http server, 那就不需要再启动一个新的 http server 



### Profiling in code

利用 runtime/pprof 包, 可以直接在代码层面进行 profile 分析.
比如开始一个 CPU 分析可以使用 <code>pprof.StartCPUProfile(io.Writer)</code> 并以 <code>pprof.StopCPUProfile()</code> 停止.
最终你可以从 io.Writer 中得到 profile.

秋豆麻得!

有一个更简单的方式可选.
Dave Cheney 开发了一个叫做 profile 的包, 它可以使得 profiling 变得更加简单.
首先你需要使用以下命令进行包的下载
```
go get github.com/pkg/profile
```

然后在代码中使用
```golang
func main(){
	defer profile.Start(profile.ProfilePath(".")).Stop()
	// do something
}
```

上述代码会将 CPU 的 profile 写入到 cpu.pprof 工作目录中.
默认是设定为 CPU 的 profiling, 不过你可以通过传递参数的形式给 profile.Start() 去得到其他的 profile.

```golang
// CPUProfile enables cpu profiling. Note: Default is CPU
defer profile.Start(profile.CPUProfile).Stop()

// GoroutineProfile enables goroutine profiling.
// It returns all Goroutines alive when defer occurs.
defer profile.Start(profile.GoroutineProfile).Stop()

// BlockProfile enables block (contention) profiling.
defer profile.Start(profile.BlockProfile).Stop()

// ThreadcreationProfile enables thread creation profiling.
defer profile.Start(profile.ThreadcreationProfile).Stop()

// MemProfileHeap changes which type of memory profiling to 
// profile the heap.
defer profile.Start(profile.MemProfileHeap).Stop()

// MemProfileAllocs changes which type of memory to profile 
// allocations.
defer profile.Start(profile.MemProfileAllocs).Stop()

// MutexProfile enables mutex profiling.
defer profile.Start(profile.MutexProfile).Stop()
```



## How to use a profile?


我们收集了 profile 数据.现在做什么呢?

pprof 是一个用来可视化与分析数据的工具.
它读取一系列由 profile.proto 格式组成的 profiling 示例,并且生成可视化报表来帮助数据分析.
它同时可支持生成文档与图表报告.


上面提到的所有方法都是在底层使用 runtime/pprof 并且这个包以 pprof 可视化工具期望的格式编写运行时分析数据。



### Write A Simple Code To Profile

为了更好地理解, 使用如下代码来进行 profile 分析.

一个给定 URL 为 /log 的 logHandler 的简单 http server.
每个请求进来时,logHandler 会声明一个 chan int 类型的变量,以及一个子协程.
子协程会去花费随机时间进行一些模拟逻辑处理,最终返回给 chan int.
在此期间, logHandler 会阻塞在 select 语句,直到子协程返回结果或者超时.

```golang
package gotour

import (
	"fmt"
	"math/rand"
	"net/http"
	_ "net/http/pprof"
	"testing"
	"time"
)

func TestLog(t *testing.T) {
	http.HandleFunc("/log", logHandler)
	_ = http.ListenAndServe(":8080", nil)
}

func logHandler(w http.ResponseWriter, r *http.Request) {
	ch := make(chan int)
	go func() {
		randInt := rand.Intn(400)
		time.Sleep(time.Duration(randInt) * time.Millisecond)
		ch <- randInt
	}()

	select {
	case status := <-ch:
		_, _ = w.Write([]byte(fmt.Sprintf("%d", status)))
	case <-time.After(200 * time.Millisecond):
		_, _ = w.Write([]byte(fmt.Sprintf("%d", 200)))
	}
}

```



接下来编译,运行代码,同时使用 hey 工具对用户发起请求.方便后面的分析.
hey 输出结果:
```
➜ ./hey_linux_amd64 -n 20000  http://localhost:8080/log

Summary:
  Total:        0.6874 secs
  Slowest:      0.0302 secs
  Fastest:      0.0000 secs
  Average:      0.0017 secs
  Requests/sec: 29096.6878
  

Response time histogram:
  0.000 [1]     |
  0.003 [17945] |■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■■
  0.006 [1300]  |■■■
  0.009 [113]   |
  0.012 [79]    |
  0.015 [58]    |
  0.018 [114]   |
  0.021 [140]   |
  0.024 [81]    |
  0.027 [149]   |
  0.030 [20]    |


Latency distribution:
  10% in 0.0001 secs
  25% in 0.0003 secs
  50% in 0.0007 secs
  75% in 0.0015 secs
  90% in 0.0031 secs
  95% in 0.0048 secs
  99% in 0.0219 secs

Details (average, fastest, slowest):
  DNS+dialup:   0.0000 secs, 0.0000 secs, 0.0302 secs
  DNS-lookup:   0.0000 secs, 0.0000 secs, 0.0000 secs
  req write:    0.0000 secs, 0.0000 secs, 0.0082 secs
  resp wait:    0.0015 secs, 0.0000 secs, 0.0286 secs
  resp read:    0.0002 secs, 0.0000 secs, 0.0085 secs

Status code distribution:
  [400] 20000 responses

```






### pprof Basics


现在是时候使用 pprof 工具来对我们服务运行生成的 profile 进行分析,查看是哪些方法开销最大.





### Install pprof Tool

你可以使用 go tool pprof 命令或者安装 pprof 来直接使用:
```
go get -u github.com/google/pprof 
```


### Run pprof On Interactive Mode

现在运行 pprof 查看对象分配
```
pprof -alloc_objects http://localhost:8080/debug/pprof/allocs
```


上述命令会打开一个简单的交互式 shell,将 pprof 命令输出来生成报告.

现在输入命令 top, 输出内存消费的大头.

```
(pprof) top
Showing nodes accounting for 764773, 83.18% of 919464 total
Dropped 27 nodes (cum <= 4597)
Showing top 10 nodes out of 31
      flat  flat%   sum%        cum   cum%
    208542 22.68% 22.68%     208542 22.68%  net/textproto.(*Reader).ReadMIMEHeader
    116261 12.64% 35.33%     513748 55.87%  net/http.(*conn).readRequest
     93846 10.21% 45.53%      93846 10.21%  net/textproto.MIMEHeader.Set
     67032  7.29% 52.82%      67032  7.29%  net/http.Header.Clone
     65537  7.13% 59.95%      65537  7.13%  net/textproto.(*Reader).ReadLine (inline)
     49153  5.35% 65.30%      49153  5.35%  context.WithCancel
     45878  4.99% 70.29%      45878  4.99%  net.(*conn).Read
     40970  4.46% 74.74%     347821 37.83%  net/http.readRequest
     39322  4.28% 79.02%      39322  4.28%  time.NewTimer
     38232  4.16% 83.18%      82123  8.93%  hhh.logHandler

```

### Dropped Nodes

top 的输出结果,你看到第三行
```
Dropped 27 nodes (cum <= 4597)
```

一个 node 表示一个对象条目,删除一个节点有助于减少信息干扰,但有时也可能会隐藏掉问题的根本原因.

在 shell 中指定 nodefraction=0 可以将所有 profile 数据输出到结果中.



## Difference Between Flat And Cumulative

在上面的输出结果中你可以看见两列:flat, cum.
Flat 意味着仅仅这个函数开销,而 cum(cumulative) 意味着当前函数与它所调用堆栈函数的开销.
为了更好的理解,我们查看以下的示例, 函数 A 的 flat time 为 4s, 而 cum time 为11s.
```golang
func A() {
   B()             // takes 1s
   DO STH DIRECTLY // takes 4s
   C()             // takes 6s
}
```




## Difference Between Allocations And Heap Profile


一个需要考虑的重点是 allocations 与 heap 的分析结构取决于 pprof 工具读取的开始时间.
Allocations 展示了程序开始时全部且包含已经被垃圾回收的分配对象数量,
但是 heap 的分析只包含存储的分配对象不包含已经被垃圾回收的字节.


这意味着你可以使用 pprof 工具来切换查看模式.
```
inuse_space: amount of memory allocated and not released yet (heap)
inuse_objects: amount of objects allocated and not released yet (heap)
alloc_space: total amount of memory allocated (regardless of released)
alloc_objects: total amount of objects allocated (regardless of released)
```


## Get An SVG Graph From The Profile

你可以基于 profile 生成 SVG 格式的图并在浏览器打开它.
如下命令会请求5s内的 CPU 分析然后在浏览器内打开.
```
pprof -web http://localhost:8080/debug/pprof/profile?seconds=5




## Run pprof Via A Web Interface

pprof 其中一个有用的特性是 web interface.
如果指定 -http 标志, pprof 会开始一个 web server 到指定的端口,通过 web server 来达到与 pprof 交互.

```
 pprof -http :9090 http://localhost:8080/debug/pprof/goroutine
```



## Monitoring Goroutines From Command Line

在 Goroutine 的分析中, 很多的协程仍然存活着,这对一个闲置的服务来说是不正常的.
所以有可能出现了协程泄露.
我们使用 grmon 工具来查看协程的状态.
grmon 是一款协程监控的命令行工具. 
以下来安装运行

```
go get -u github.com/bcicen/grmon
grmon -host localhost:8080
```

output:
```
  1    chan receive        testing.(*T).Run(0xc000001b00, 0x73325f, 0x7, 0x74c4f0, 0x48ef66)
  6    IO wait             internal/poll.runtime_pollWait(0x7f1f1c18f838, 0x72, 0x0)
  7    running             runtime/pprof.writeGoroutineStacks(0x792d20, 0xc0000f20e0, 0x30, 0x702ae0)
  21   chan send           hhh.logHandler.func1(0xc000154060)
  25   chan send           hhh.logHandler.func1(0xc0001541e0)
  27   chan send           hhh.logHandler.func1(0xc0001542a0)
  41   chan send           hhh.logHandler.func1(0xc0002e6000)
  47   chan send           hhh.logHandler.func1(0xc0002e60c0)
  60   chan send           hhh.logHandler.func1(0xc000022660)

 total: 724 
```

看起来是因为 logHandler 函数阻塞了 chan send 操作,
出现原因是由于发送数据给一个不确定的 channel.
通过修改 ch 为一个带缓存大小为 1 的 channel 解决这个问题
```golang
ch := make(chan int, 1)
```



















```





 
