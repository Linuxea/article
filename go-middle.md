# Learn Go Middlewares by Examples


Middlewares are one of the most important concepts in backend engineering. They are standalone, reusable pieces of software that link different systems together.

中间件是后台工程中最重要的概念之一。

它们是独立的，可重用的软件组件，将不同的系统连接起来。


In web development, it is common to place one or many middlewares between the client and server. This essentially creates a bridge between data and user interfaces.


在 web 开发中，client 与 server 之间普通存在着一个或者多个中间件。这潜在地为数据与 UI 搭建了一座桥梁。

Each middleware acts independently on an HTTP request or response. The output of one middleware can be the input of another middleware. This forms a chain of middlewares.

中间件独立于 Http 的请求与响应。

一个中间件的输出可以作为另外一个中间件的输入。

通过这种方式组成了一条中间件链。

Why middlewares? In web applications, middlewares allow us to centralize and reuse common functionalities on each request or response. For example, you can have a middleware that logs every HTTP request.


为什么要中间件？

在 web 应用中，中间件的应用可以让我们集中和重复每个请求与响应的的通用功能。比如最常见的为每个请求进行日志记录。


To design Go middlewares, there are certain rules and patterns that we have to follow. This article will teach you the concepts of a Go middleware and create two of them for a simple application!


在设计中间件这一方面，有一些明确的规则与模式需要我们去遵守。

这篇文章将会通过一个简单的应用来演示这些。

## Getting Started

```go
package main

import (
	"log"
	"math/rand"
	"net/http"
	"time"
)

func main() {
	// Initialize a new ServeMux
	mux := http.NewServeMux()
	// Register home function as handler for /home
	mux.HandleFunc("/home", home)
	// Run HTTP server with custom ServeMux at localhost:4000
	err := http.ListenAndServe(":8080", mux)
	// Exit if any errors occur
	log.Fatal(err)
}

// Handles HTTP requests to /home
func home(w http.ResponseWriter, r *http.Request) {
	// Writes a byte slice with the text "Welcome to Go Middleware"
	// in the response body
	time.Sleep(time.Second*time.Duration(rand.Intn(2)) + 1)
	w.Write([]byte("Welcome to Go Middleware"))
}
```


在上述代码中，我们创建了一个新的非默认的 `ServeMux`， 添加了路由规则 `/home` 映射到处理函数 `home`,
随后将这个  `ServeMux` 注册监听到 `8080` 端口。

将代码运行起来并在终端进行请求会得到如下：
```bash
➜  go_middle go run . &
[1] 28901
➜  go_middle curl localhost:8080/home
Welcome to Go Middleware
```


## Handler and ServeHTTP

Before we start creating Go middleware, it is good for us to know some theories. In particular, what exactly is a handler in Go?


在我们开始创建中间件前，理论一些理论知识是有益的。

特别是，在 Go 中的 `handler` 是什么。

In Go, a handler is just an object (struct) that satisfies the http.Handler interface.

在 Go 中，`handler` 是满足 `http.Handler` 接口定义的一个对象或者结构体


```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
Meanwhile, a ServeMux object (be it default or custom) is also a http.Handler . It executes its ServeHTTP method when it receives an HTTP request, which matches the URL request path with the appropriate handler.

If there is a match, ServeMux proceeds to call the ServeHTTP method of the handler and passes the request to it. The handler then executes its route handling logic and returns a response.


In a way, you can think of Go’s application routing as a chain of handlers with ServeHTTP methods called one after another.


通过 `ListenAndServe` 方法签名
```go
func ListenAndServe(addr string, handler Handler) error {
... ...
}
```

我们知道 `ServeMux` 本身就是一个 `http.Handler` .

当它接收到一个 http 请求时，会根据 URL 找到匹配的处理器，比如  `home`, `home` 方法的定义同样符合 `http.Handler` 的接口声明，通过  `http.HandlerFunc(...)` 可以将其转化为 `handler`， 所以路由处理同样也是 `http.Handler`.

在 URL 匹配时，`ServeMux` 会调用路由处理器的 `ServeHTTP` 方法并将请求传递给它， 处理器在随后进行它的逻辑处理并响应。


总之，你可以认为 Go 的 web 应用路由是通过一系列的 `http.Handler`  与方法 `ServerHTTP` 依次调用来实现的。



Hence, to fit into the chain, a Go middleware must behave like a handler. It performs some logic before passing a request to the next handler by calling the handler’s ServeHTTP method!

因此，为了适应进入这一条链中，Go 中间件同样也需要扮演 `http.Handler`  的角色。

它执行一些特定的逻辑与调用下一个 `http.Handler` 的 `ServeHTTP` 的方法。



## The Pattern of Go Middleware

举例子

我们需要一个 `traceIdHandlerProxy` 的中间件，顾名思义就是在请求与响应头添加 `traceId` 来做分布式链接追踪的基础工作.

### constructor

第一种方式使用构造器

```go
// trace id 代理 http.Handler
type traceIdMiddle struct {
	next http.Handler
}

// satisfy http.Handler interface
func (h *traceIdMiddle) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
	// traceIdHandlerProxy logic
	traceId := time.Now().Format("20060102150405")
	r.Header.Set("trace_id", traceId)
	rw.Header().Set("trace_id", traceId)

	// forward to next http.Handler
	h.next.ServeHTTP(rw, r)
}
```

创建出一个新的结构体 `traceIdHandlerProxy`，添加方法 `ServeHTTP(ResponseWriter, *Request)` 来满足 `http.Handler` 接口的要求。

这样就成为了一个 `http.Handler`.

优点：
- 简单直观，面向对象（结构体）的开发者很熟悉
缺点：
- 需要创建大量的中间件结构体


### higher function 


```golang
func traceIdMiddle(next http.Handler) http.Handler {
	f := func(rw http.ResponseWriter, r *http.Request) {
		// traceIdHandlerProxy logic
		traceId := time.Now().Format("20060102150405")
		r.Header.Set("trace_id", traceId)
		rw.Header().Set("trace_id", traceId)

		// forward to next http.Handler
		next.ServeHTTP(rw, r)
	}
	return http.HandlerFunc(f)
}
```


优点：
- 代码量少，灵活，使用高阶函数符合函数式编程规范
缺点：
- 无



接下来的示例都会以高阶函数的方式来完成例子


A lot is going on in the snippet above. Let’s break it down.
goMiddleware is a function that accepts a next parameter of type http.Handler and returns another http.Handler .
A function f is created inside goMiddleware . It takes in two parameters of type http.ResponseWriter and *http.Request , which are the signatures of a typical handler function.
The middleware logic is contained inside f .
f passes the request to the next handler by calling the handler’s ServeHTTP method.
f is transformed and returned as a http.Handler with the http.HandlerFunc adapter.
The returned f forms a closure over the next handler. Hence, it can still access the local next variable even after it is returned by goMiddleware .



上面的代码片段包含很多信息，一起来看看：
- traceIdMiddle 是一个接收参数为 `http.Handler`，返回参数也为 `http.Handler` 的函数。
- 中间件内部创建了一个符合 `handler` 签名的函数 `f`， 它接收两个参数，分别是 `http.Responsewriter`  与 `*http.Request` ，中间件的逻辑也包含在函数 `f` 内。
- 函数 `f` 内部通过调用下一个 `http.Handler`  的 `ServerHTTP` 来传递 http 请求。
- 函数 `f` 通过 `http.HandlerFunc` 函数转换与适配为 `http.Handler` 并返回。
- `f` 形成了一个闭包，因为在其被返回后依然能够访问到变量 `next` 。 

The main takeaway is that a Go middleware is a function that accepts the next handler in the request chain as a parameter. It returns a handler that performs some logic before executing the next handler in the chain.

主要内容是 Go 中间件是一个函数，它接受请求链中的下一个 `http.Handler` 作为参数。 它返回一个 `http.Handler`，该处理程序在执行链中的下一个处理程序之前或者之后执行一些逻辑。


## Creating Go Middlewares

