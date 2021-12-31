

> golang 中的 context 包在 api 交互与慢处理时可以派上用场，尤其是在服务于 Web 请求的生产级系统中。在这种情况下，你可能想要通知所有的协程停止工作并返回。这篇是关于如何在你们的项目中使用 context 包的基础教程，其中包含一些最佳实践与陷阱。




## Prerequisites

在了解 context 包之前，有两个概念你应该知晓：
- goroutine 协程
- channel 通道

在进入 context 之前，会先尝试介绍这些内容。如果你已经熟悉这两个概念，可以跳过直接进入 context 介绍。


### Goroutine

来自官方文档的介绍：“协程是轻量级的执行线程”
协程比线程更加轻量级，因此对它们的资源管理紧急程度相对较低


```go
package main

import "fmt"

//function to print hello
func printHello() {
	fmt.Println("Hello from printHello")
}

func main() {
	//inline goroutine. Define a function inline and then call it.
	go func() { fmt.Println("Hello inline") }()
	//call a function as goroutine
	go printHello()
	fmt.Println("Hello from main")
}

```


运行上面代码，你可能只会看到打印出“Hello from main”，这是因为启动的协程在 main 函数退出时还没有结束

为了确认 main 函数等待协程结束，需要一些方法来让协程告诉 main 已经执行完成。这就是 channel 通道可以帮得上我们的地方

### Channel

这是协程之间用来通信的通道。channel 被用来从一个协程向另外一个协程传递结果,异常，或者任何类型的信息

channel 是有类型的。可以让一个 int 类型的通道来接收整数或者让一个 error 类型的通道来接收错误

假设存在一个 int 类型的通道 ch，向这个通道发送消息的语法是： ch <- 1
从这个通道接收消息的语法是：var := <- ch，表示从通道 ch 中接收消息并存储在变量 var 中

接下来的例子会说明通道的用法：确保协程执行结束并返回值到 main

>Wait groups 也可以用来做协程之间的同步，不过因为我们是在 context 包这一章节

```go
package main

import "fmt"

//prints to stdout and puts an int on channel
func printHello(ch chan int) {
	fmt.Println("Hello from printHello")
	//send a value on channel
	ch <- 2
}

func main() {
	//make a channel. You need to use the make function to create channels.
	//channels can also be buffered where you can specify size. eg: ch := make(chan int, 2)
	//that is out of the scope of this post.
	ch := make(chan int)
	//inline goroutine. Define a function and then call it.
	//write on a channel when done
	go func() {
		fmt.Println("Hello inline")
		//send a value on channel
		ch <- 1
	}()
	//call a function as goroutine
	go printHello(ch)
	fmt.Println("Hello from main")

	//get first value from channel.
	//and assign to a variable to use this value later
	//here that is to print it
	i := <-ch
	fmt.Println("Received ", i)
	//get the second value from channel
	//do not assign it to a variable because we dont want to use that
	<-ch
}

```



## Context

一种思考 context 的方式是它允许你传入一个上下文到你的程序中
Context 就像一个超时，或者截止日期，或者一个通道来表明协程的停止与返回
比如，你正在处理一个 web 请求或者运行一个系统命令，通常比较好的主意是生产环境设定一个超时时间
因为你所依赖的 api 如果运行缓慢，你不会希望它在你的系统中重复，因为它可能提高负载与降低所有请求处理性能，造成系统级联影响
而避免这种情况，就需要一个超时，拥有截止时间的 context 上场了


### Creating context

context 包提供创建与派生 context 的几种方式：

#### context.Background() ctx Context

这个方法返回一个空的 context。它应该在高层次的地方所使用到，比如 main 函数或者高层次的请求处理中
它可以被用来派生其他的 context ，稍后我们会讨论到

```golang
ctx, cancel := context.Background()
```

#### context.TODO() ctx Context

这个方法同时创建了一个空的 context。它也同样应该只会应用到高层次的地方，或者一个你不确定要使用什么上下文或函数尚未更新以接收上下文的时候
这表明了代码作者会在将来为该功能添加上下文

```golang
ctx, calcel := context.TODO()
```

有趣的是，查看 go 源码，会发现它与 context.Background() 是一致的
```golang
var (
	background = new(emptyCtx)
	todo       = new(emptyCtx)
)

```

区别在于静态分析工具可以用来验证 context 是否被正确传递，这是一个重要的细节，因为静态分析工具可以帮助尽早发现潜在的错误


#### context.WithValue(parent Context, key, val interface{})

这个方法接收一个 context 参数并返回一个派生 context，其中 val 与 key 相关联并与上下文流经上下文树。这意味着一旦你获得一个带有值的上下文，任何由此派生的上下文都会得到这个值。
但是并不推荐使用 context.WithValue 来传递重要的参数，而是应该在方法的参数中显式的传递。

```golang
ctx := context.WithValue(context.Background(), key, "test")
```

#### context.WithCalcel(parent Context)(ctx Context, calcel CalcelFunc)
这里开始变得有趣一点。这个方法从传递进来的父上下文创建了一个新的派生上下文。父上下文可能是 background 上下文或者从外部传递进方法的上下文。

它返回的是一个派生上下方与一个取消函数。只有创建这个上下文的函数才应该调用这个取消函数来取消这个上下文。如果你想也可以将这个取消函数传递出去，但是这是非常不推荐的。这可能会导致取消函数的调用者没有意识到取消上下文所带来的下游影响，可能会存在一些由此派生的上下文造成程序出现预期外的行为。

总而言之，永远不要把取消函数传递出去。

```golang
ctx, calcel := context.WithCancel(context.Background())
```


#### context.WithDeadline(parent Context, d time.Time)(ctx Context, cancel CalcelFunc)

这个方法调用从父上下文返回了一个派生上下文，当截止日期到达时或者被调用了取消函数，将会被取消上下文。
比如，你可以创建一个在将来某个明确时间自动取消的上下文，然后把它传递到子函数中，当上下文到了截止日期取消时，所有得到这个上下文的函数都会收到通知去停止工作并返回。

```golang
ctx, cancel := context.WithDeadline(context.Background(), time.Now().Add(2 * time.Second))
```


#### context.WithTimeout(parent Context, d time.Duration)(ctx Context, cancel CancelFunc)

这个方法跟 `context.WithDeadline` 相似。区别在于它接收的是一个 `time.Duration` 参数而不是 `time.Time` 对象。
这个方法返回一个派生上下文，会在被调用取消函数时或者超时时间截止时被取消。

```golang
ctx , cancel := context.WithTimeout(context.Background(), time.Duration(150) * time.Millisecond)
```



### Accepting and using contexts in your functions

现在我们知道了如何创建上下文(使用 Background 和 TODO),以及如何派生上下文(WithValue, WithCancle, Deadline and Timeout).现在我们来讨论下如何使用它们.
在下面的例子中, 你可能看见一个带有接收 context 参数的方法, 方法内部启动了一个协程, 等待着协程的返回或者 context  的取消. select 表达式帮助我们选择哪个先发生并返回.

`<- ctx.Done()` 一旦  context  的 Done 通道关闭,  `case <- ctx.Done():` 将会被选择.
这种情况如果发生, 函数应该停止工作并且准备返回.
这意味着你应该关闭打开的管道,闲置的资源并且从函数中返回.在某些情况下，释放资源可能会阻止返回，例如进行一些挂起的清理等。您应该在处理上下文返回时注意任何此类可能性。 


cancel
```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"time"
)

func main() {

	// create a context with `context.Background()`
	background := context.Background()
	// create a derived context with `context.WithCancel`
	cancelCtx, cancelFunc := context.WithCancel(background)

	go func() {
		time.Sleep(time.Duration(rand.Intn(5)) * time.Second)
		// invoke cancel func after seconds
		cancelFunc()
	}()

	info := getUserInfo(cancelCtx, 2)
	fmt.Println(info)

}

// mock get user info
func getUserInfo(ctx context.Context, userId uint) uint {

	ch := make(chan int)
	go func() {
		time.Sleep(time.Duration(rand.Intn(30)) * time.Second)
		// get user info successfully
		ch <- 1
	}()

	select {
	case <-ctx.Done():
		// the parent context is canceled
		return 0
	case <-ch:
		// get user info successfully
		return userId
	}

}

```

deadline
```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"time"
)

func main() {

	// create a context with `context.Background()`
	background := context.Background()
	// create a derived context with `context.WithTimeout`
	// ignore cancel func
	timeoutCtx, _ := context.WithDeadline(background, time.Now().Add(time.Second*1))
	info := getUserInfo(timeoutCtx, 2)
	fmt.Println(info)

}

// mock get user info
func getUserInfo(ctx context.Context, userId uint) uint {

	ch := make(chan int)
	go func() {
		duration := time.Duration(rand.Intn(3)) * time.Second
		time.Sleep(duration)
		// get user info successfully
		ch <- 1
	}()

	select {
	case <-ctx.Done():
		// the parent context is canceled
		return 0
	case <-ch:
		// get user info successfully
		return userId
	}

}

```


timeout
```go
package main

import (
	"context"
	"fmt"
	"math/rand"
	"time"
)

func main() {

	// create a context with `context.Background()`
	background := context.Background()
	// create a derived context with `context.WithTimeout`
	timeoutCtx, _ := context.WithTimeout(background, time.Second*1)
	info := getUserInfo(timeoutCtx, 2)
	fmt.Println(info)

}

// mock get user info
func getUserInfo(ctx context.Context, userId uint) uint {

	ch := make(chan int)
	go func() {
		duration := time.Duration(rand.Intn(3)) * time.Second
		time.Sleep(duration)
		// get user info successfully
		ch <- 1
	}()

	select {
	case <-ctx.Done():
		// the parent context is canceled
		return 0
	case <-ch:
		// get user info successfully
		return userId
	}

}

```




## Best Practices
- context.Background 应该被用在程序的最高层次,作为其他所有派生上下文的根
- context.TODO 应该被用在你不确定将来会被更新使用上下文的地方
- 使用上下文取消是建议性的，函数可能需要时间来清理和退出
- context.Value 应该很少使用，它永远不应该用于传递可选参数.这使得 API 隐晦并可能引入错误.相反，这些值应该作为参数传入
- 不要在结构中存储上下文，在函数中显式传递它们，而且最好是作为第一个参数
- 永远不要传递 nil 上下文，相反，如果您不确定要使用什么，请使用 TODO
- Context 结构没有取消方法，因为只有派生上下文的函数才能取消它



## 参考

- [1] [Understanding the context package in golang](http://p.agnihotry.com/post/understanding_the_context_package_in_golang/)
- [2] [context package](https://pkg.go.dev/context)