# Learn Go Middlewares by Examples

中间件是后台工程中最重要的概念之一。

它们是独立的，可重用的软件组件，将不同的系统连接起来。

在 web 开发中， `client` 与 `server` 之间普通存在着一个或者多个中间件。潜在地为数据与 UI 搭建了一座桥梁。

中间件独立于 Http 的请求与响应。

一个中间件的输出可以作为另外一个中间件的输入。

通过这种方式组成了一条中间件链。

为什么要中间件？

在 web 应用中，中间件的应用可以让我们集中和重复每个请求与响应的的通用功能。比如最常见的为每个请求进行日志记录。

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
	time.Sleep(time.Second*time.Duration(rand.Intn(5)) + 1)
	w.Write([]byte("Welcome to Go Middleware"))
}
```


在上述代码中，我们创建了一个新的并且非默认的 `ServeMux`， 添加了路由规则 `/home` 映射到处理函数 `home`,
随后将这个  `ServeMux` 注册监听到 `8080` 端口。

将代码运行起来并在终端进行请求会得到如下：
```bash
➜  go run . &
[1] 28901
➜  curl localhost:8080/home
Welcome to Go Middleware
```


## Handler and ServeHTTP

在我们开始创建中间件前，理论一些理论知识是有益的。

特别是，在 Go 中的 `handler` 是什么。

在 Go 中，`handler` 是满足 `http.Handler` 接口定义的一个对象或者结构体。


```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

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



上面的代码片段包含很多信息，一起来看看：
- traceIdMiddle 是一个接收参数为 `http.Handler`，返回参数也为 `http.Handler` 的函数。
- 中间件内部创建了一个符合 `handler` 签名的函数 `f`， 它接收两个参数，分别是 `http.Responsewriter`  与 `*http.Request` ，中间件的逻辑也包含在函数 `f` 内。
- 函数 `f` 内部通过调用下一个 `http.Handler`  的 `ServerHTTP` 来传递 http 请求。
- 函数 `f` 通过 `http.HandlerFunc` 函数转换与适配为 `http.Handler` 并返回。
- `f` 形成了一个闭包，因为在其被返回后依然能够访问到变量 `next` 。 

主要内容是 Go 中间件是一个函数，它接受请求链中的下一个 `http.Handler` 作为参数。 它返回一个 `http.Handler`，该处理程序在执行链中的下一个处理程序之前或者之后执行一些逻辑。


## Creating Go Middlewares

现在我们了解中间件正确的概念与理论，接下来的事情会变得更加简单。

在这一节，我们尝试创建多个不同的中间件，一个是记录 http 的请求日志，一个是添加添加基础的安全 header 信息，另外一个是统计业务处理的时间。


```golang
package main

import (
	"fmt"
	"net/http"
	"time"
)

func logRequestMiddleware(handler http.Handler) http.Handler {
	return http.HandlerFunc(func(rw http.ResponseWriter, r *http.Request) {

		fmt.Printf("LOG %s - %s %s %s\n", r.RemoteAddr, r.Proto, r.Method, r.URL)

		// continue handle
		handler.ServeHTTP(rw, r)

	})
}

func secureHeadersMiddleware(handler http.Handler) http.Handler {
	return http.HandlerFunc(func(rw http.ResponseWriter, r *http.Request) {

		rw.Header().Set("X-XSS-Protection", "1; mode-block")
		rw.Header().Set("X-Frame-Options", "deny")

		// continue handle
		handler.ServeHTTP(rw, r)

	})
}

func handleTimeLogMiddle(handler http.Handler) http.Handler {
	return http.HandlerFunc(func(rw http.ResponseWriter, r *http.Request) {

		now := time.Now()

		// continue handle
		handler.ServeHTTP(rw, r)

		duration := time.Since(now).Milliseconds()
		fmt.Printf("请求处理时间: %d(ms)", duration)

	})
}

```

`logRequestMiddleware` 记录了网络地址，协议版本，与请求方法，请求 URL 到标准输出中。这些信息在 `http.Request` 是可获取的。

`secureHeadersMiddleware` 在响应中设置了两个安全头部信息来防御对抗 XSS 等攻击。

`handleTimeLogMiddle` 在下一个 `handler` 执行前记录了开始时间，在处理完成后记录结束时间，以此来计算总共花费处理时长，这可以很方便找出处理时长较高的逻辑。


但是，在我们注册这些中间件之前，很重要的一点是要明确知道中间件位置的不同可能对我们应用行为造成的影响。

特别是，如果我们想要一个中间作用于每个 `HTTP` 请求，那么我们应该将它放置在 `ServeMux` 之前，换种方式说，我们需要将 `ServeMux` 作为下一个待处理 `http.Handler` 传递给我们的中间件。

另一方面，如果我们想要中间件作用于特定的路由，我们需要将它放置在 `ServeMux` 之后，将其作为参数传递给我们的中间件来实现这一点。

在这个例子中，我们需要将 `logHttpRequestMiddle` 与 `securityHeaderMiddle` 中间件作用于每一个 HTTP 请求。因此两个中间件都应该放置在 `ServeMux` 之前。


```golang
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
	err := http.ListenAndServe(":8080", logHttpRequestMiddle(securityHeaderMiddle(mux)))
	// Exit if any errors occur
	log.Fatal(err)
}

// Handles HTTP requests to /home
func home(w http.ResponseWriter, r *http.Request) {
	// Writes a byte slice with the text "Welcome to Go Middleware"
	// in the response body
	time.Sleep(time.Second*time.Duration(rand.Intn(5)) + 1)
	w.Write([]byte("Welcome to Go Middleware"))
}
```




另外我们只需要统计 `home` 处理器的执行时长，因此我们需要 `handleTimeLogMiddle` 中间件放置在 `ServeMux` 之后。
```golang
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
	mux.HandleFunc("/home", func(rw http.ResponseWriter, r *http.Request) {
		handleTimeLogMiddle(http.HandlerFunc(home)).ServeHTTP(rw, r)
	})
	// Run HTTP server with custom ServeMux at localhost:4000
	err := http.ListenAndServe(":8080", logHttpRequestMiddle(securityHeaderMiddle(mux)))
	// Exit if any errors occur
	log.Fatal(err)
}

// Handles HTTP requests to /home
func home(w http.ResponseWriter, r *http.Request) {
	// Writes a byte slice with the text "Welcome to Go Middleware"
	// in the response body
	time.Sleep(time.Second*time.Duration(rand.Intn(5)) + 1)
	w.Write([]byte("Welcome to Go Middleware"))
}


```


## Quick Testing

通过 `curl` 命令来检查中间件工作情况。

- 启动 HTTP 服务器并以后台方式运行
- curl 命令请求 `/home` 路由地址 
  - `-X` 选项用来指定请求的方式为 `GET` 
  - `-i` 选项用来打印响应 `header` 信息

```bash
➜  go-middle go run . &
[1] 30145

➜  go-middle curl -X GET -i localhost:8080/home
LOG [::1]:53846 - HTTP/1.1 GET /home
请求处理时间: 1004(ms)HTTP/1.1 200 OK
X-Frame-Options: deny
X-Xss-Protection: 1; mode-block
Date: Mon, 10 Jan 2022 15:11:00 GMT
Content-Length: 24
Content-Type: text/plain; charset=utf-8

Welcome to Go Middleware
```


我们可以看到 `logHttpRequestMiddle` 中间件打印了请求日志 
```bash
LOG [::1]:53846 - HTTP/1.1 GET /home
```

`handleTimeLogMiddle` 打印出了执行时长
```bash
请求处理时间: 1004(ms)HTTP/1.1 200 OK
```

`securityHeaderMiddle` 设置了头部信息
```bash
X-Frame-Options: deny
X-Xss-Protection: 1; mode-block
Date: Mon, 10 Jan 2022 15:11:00 GMT
Content-Length: 24
Content-Type: text/plain; charset=utf-8
```


## Final Thoughts

走到这里意味着你对 Go 中间件编写已经有了初步的了解。

你也可以发现通过这种方式，当中间件链数量太多，代码并不具备很好的拓展性，从而难以维护。

幸运的是，已经有第三方包来帮助我们管理中间件。其中有一些比较出名，可以去了解一下。


## 参考

- [1] [Learn Go Middlewares by Examples](https://medium.com/geekculture/learn-go-middlewares-by-examples-da5dc4a3b9aa)
- [2] [Using Dependency Inversion in Go](https://medium.com/itnext/using-dependency-inversion-in-go-31d8bf9b3760)
