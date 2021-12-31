# Introducing the Go Race Detector

## Introduction

`race condition` (竞争条件)存在于绝大多数隐蔽且难以捉摸的编程错误之中。

它们通常会在代码被部署到正式环境很长一段时间后制造不稳定与诡异的失败。

尽管 Go 的并发机制使得编写简洁的并发代码变得容易，但是它们却不能避免竞争条件。

谨慎，勤勉,与测试不可或缺，合适的工具也同样重要。

我们很高兴宣布 Go1.1 包含了 race detector （竞争检测器）, 它是一个用来检测 Go 代码中竞争条件的工具。
目前它支持的包括 64 位 X84 架构处理器的 linux, OS X, 与 Windows 系统.



## How it works

竞争检测器已经集成到 go 的工具链中.

当设置命令行标识 `-race` 时, 代码关于所有内存访问的编译指令都会被记录何时以及怎样的内存访问方式, 运行时库会监控对共享变量的非同步访问.

当检测到这种行为时,会打印出警告信息.

因为其本身的设计,竞争检测器只能检测出由运行时代码触发的竞争条件, 这意味着在实际负载下运行包含竞争检测器的二进制文件是至关重要的.

然而,包含竞争检测器的二进制文件会使用十倍的 CPU 与内存,所以让竞争检测器一直运行是不切实际的.

摆脱这种困境的方法之一是在启用竞争检测器的情况下运行测试.

负载测试与集成测试是不错的方案,因为它们都倾向于执行代码中的并发部分.

另外一种方法是使用生产线上的工作负载，在运行的多台服务器上部署一台启动竞争检测的实例.


## Using the race detector

竞争检测器已经完全集成到 Go 工具链当中.

为了将竞争检测器与代码完成构建, 仅仅需要添加 `-race` 标记到命令行中.

```
$ go test -race mypkg    // test the package
$ go run -race mysrc.go  // compile and run the program
$ go build -race mycmd   // build the command
$ go install -race mypkg // install the package
```


为了自己能够尝试使用竞争检测器, 你可以拷贝如下代码来 `racy.go` 文件中:

```golang
package main

import "fmt"

func main() {
    done := make(chan bool)
    m := make(map[string]string)
    m["name"] = "world"
    go func() {
        m["name"] = "data race"
        done <- true
    }()
    fmt.Println("Hello,", m["name"])
    <-done
}
```



然后开启竞争检测器运行:

```
$ go run -race racy.go
```




## Examples

以下是竞态检测器捕获的实际问题的示例。



### Example 1: Timer.Reset


第一个是简化版的由竞争检测器发现的一个 bug.

它使用了一个定时器,在1秒内的随机时间打印一条信息, 然后在五秒钟内重复这个逻辑。

使用了 `time.AfterFunc` 为第一条消息创建定时器然后使用 `Reset` 方法复用定时器调度下一条消息

```golang
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	start := time.Now()
	var t *time.Timer
	t = time.AfterFunc(randomDuration(), func() {
		fmt.Println(time.Now().Sub(start))
		t.Reset(randomDuration())
	})
	time.Sleep(5 * time.Second)
}

func randomDuration() time.Duration {
	return time.Duration(rand.Int63n(1e9))
}
```


这看起来像是合理的代码，不过在某些场景下会以一种令人惊讶的方式失败。


```
panic: runtime error: invalid memory address or nil pointer dereference
[signal 0xb code=0x1 addr=0x8 pc=0x41e38a]

goroutine 4 [running]:
time.stopTimer(0x8, 0x12fe6b35d9472d96)
    src/pkg/runtime/ztime_linux_amd64.c:35 +0x25
time.(*Timer).Reset(0x0, 0x4e5904f, 0x1)
    src/pkg/time/sleep.go:81 +0x42
... ...
```


这里发生了什么？

开启竞争检测器来运行程序会使得情况更加明朗。

```
➜  go run -race .
948.805121ms
==================
WARNING: DATA RACE
Read at 0x00c000136018 by goroutine 8:
  main.main.func1()
      /home/linuxea/Desktop/code/dlv-demo/racy.go:14 +0x126

Previous write at 0x00c000136018 by main goroutine:
  main.main()
      /home/linuxea/Desktop/code/dlv-demo/racy.go:12 +0x194

Goroutine 8 (running) created at:
  time.goFunc()
      /home/linuxea/soft/go/src/time/sleep.go:180 +0x51
==================
1.034312152s
1.701386981s
1.937044592s
2.224530328s
2.774808307s
3.40869678s
3.74132969s
3.924653288s
4.405108767s
Found 1 data race(s)
exit status 66
```


竞争检测器展示了问题所在：来自不同的协程以非同步的方式对变量 t 进行了读写操作。
如果初始化 timer 的 duration 非常小，定时器的函数可能在主协程为 t 分配值之前就触发，对 t.Reset 的调用则为对 `nil` 的 调用。


为了修复这个竞争条件，我们可以修改代码只通过主协程来读写变量 `v`

```golang
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func main() {
	start := time.Now()
	reset := make(chan bool)
	var t *time.Timer
	t = time.AfterFunc(randomDuration(), func() {
		fmt.Println(time.Now().Sub(start))
		reset <- true
	})
	for time.Since(start) < 5*time.Second {
		<-reset
		t.Reset(randomDuration())
	}
}

func randomDuration() time.Duration {
	return time.Duration(rand.Int63n(1e9))
}
```

修改之后的主协程完全负责变量 t 的设置与重置，一个新的重置的 channel 以一种线程安全的方式表达重置 timer 的需要。

另外一个更加简单但是效率低点的方法是避免重复使用定时器。

还可以通过锁或读写锁的方式来对共享资源访问。

具体的不再演示。


## Conclusions

竞争检测器是用来检测并发程序正确性的强有力工具。

它不会发出误报，所以对它的警告信息需要严肃对待。

但它也只是跟你的测试一样优秀的工具。（协同合作而不是相互取代）

你必须确保它们彻底执行代码中的并发属性，以便让检测竞争器来做好它的工作。

你还在等待什么？从今天开始在你的代码运行 `go test -race`吧！




## 参考

- [1] [Introducing the Go Race Detector](https://go.dev/blog/race-detector)
- [2] [Race conditions in Golang](https://medium.com/trendyol-tech/race-conditions-in-golang-511314c0b85)



