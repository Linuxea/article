


## Using Dependency Inversion in Go




### What is Dependency Inversion?


依赖倒置是高层逻辑不应该依赖于低层的实现的一种理念.

比如应用中业务逻辑不应该关心数据来自 amazon 的 AWS 抑或谷歌的云存储.

我们应该在能力在不破坏程序的情况下轻易地切换底层的实现.

这使得我们能够更加稳定地对抗变化.

我们也可以通过切换应用程序更加容易测试一些实现来使得我们的应用程序可测试能力更强.



### How is this done in Go?

在 go 语言中, 接口(interface) 赋予了我们使用依赖倒置的能力.

我们能够在代码中使用不同的实现,前提是它们都实现了接口的定义.

我们使依赖注入的方式来告诉应用程序使用哪一种实现.






## Worked Example

为了阐述这在 Go 中的工作方式,我们构建一个提供随机引用的 API.
如下是代码:
```golang
package main

import (
	"encoding/json"
	"io/ioutil"
	"net/http"
)

func main() {

	s := &http.Server{
		Addr:    ":8080",
		Handler: &kanye{},
	}

	s.ListenAndServe()

}

type kanye struct {
}

func (*kanye) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	resp, err := http.Get("https://api.kanye.rest/")
	if err != nil {
		_, _ = w.Write([]byte("error getting kanye quote"))
		return
	}

	if resp == nil || resp.StatusCode != http.StatusOK {
		_, _ = w.Write([]byte("bad kanye response"))
		return
	}

	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		_, _ = w.Write([]byte("bad kanye body"))
		return
	}

	defer resp.Body.Close()

	var quote quote
	err = json.Unmarshal(body, &quote)
	if err != nil {
		_, _ = w.Write([]byte("could not marshal kanye"))
		return
	}

	_, _ = w.Write([]byte(quote.Quote))

}

type quote struct {
	Quote string `json:"quote"`
}

```

代码发起 http 请求到 https://api.kanye.rest, 提供引用的返回并进行响应是否有效.

当你调用这个 handler, 你会得到一句引用.
```bash
➜  ~ curl http://localhost:8080
I honestly need all my Royeres to be museum quality... if I see a fake Royere Ima have to Rick James your couch
```


然而,如果我们想要对代码进行测试,会发现目前无法在不发起真实 http 调用的条件下完成这一工作.

我们也没有办法测试出依赖返回错误响应时会发生什么.

从这里开始,我们会开始使用依赖倒置.



首先我们需要创建一个接口, 定义了我们所依赖的行为.
通过这种方式我们可以稍后有能力进行不同实现的切换:
```golang
type Client interface {
	get(string) (*http.Response, error)
}
```

任何结构体只要拥有名称为 `get`的 方法以及一个 `string` 参数,并且返回 net/http 包下的 Response 指针及 error, 都是对接口 `Client` 的实现.
这也是我们所感兴趣的行为.


现在我们需要一种将依赖项注入程序的方法,使其与我们使用的 Client 接口的任何实现无关.

Go 中有很多依赖注入的方法, 我将在下面概述一些:


### Higher Order Function

我们可以创建一个高阶函数来返回原始的 handler 方法(ServeHTTP(ResponseWriter, *Request))


```golang
func kanye(client Client) func(w http.ResponseWriter, r *http.Request) {

	return func(w http.ResponseWriter, r *http.Request) {
		resp, err := client.get("https://api.kanye.rest/")
		if err != nil {
			_, _ = w.Write([]byte("error getting kanye quote"))
			return
		}
	}
}
```


我们的 http 依赖项然后通过高阶函数进入iyty

```golang

func kanye(client Client) func(w http.ResponseWriter, r *http.Request) {

	return func(w http.ResponseWriter, r *http.Request) {
		resp, err := client.get("https://api.kanye.rest/")
		if err != nil {
			_, _ = w.Write([]byte("error getting kanye quote"))
			return
		}
	}

}
```



### Constructor


我们可以创建一个 `handlers` 结构体,同时提供一个方法来提供 `client` 的实现.

```golang
package handlers

type handlers struct {
	client Client
}

func NewHandlers(client Client) *handlers {
	return &handlers{client: client}
}
```



通过制作接收者为 handlers 的 `Kanye` 方法,我们可以访问到 `client` 字段

```golang
func (h *handlers) Kanye(w http.ResponseWriter, r *http.Request) {
	_, err := h.client.get("https://api.kanye.rest/")
	if err != nil {
		_, _ = w.Write([]byte("error getting kanye quote"))
		return
	}
}
```

我们可以通过使用正确的依赖来构建 handlers 结构体,并将其注入.

这种方法的一个缺点是，当大量使用时，它会使您在主文件中构造大量结构。

```golang
func main() {
	r := mux.NewRouter()
	h := handlers.NewHandlers(http.DefaultClient)
	r.HandleFunc("/kanye", h.Kanye)
	http.ListenAndServe(":2024", r)
}

```


### Using Options

另外一个方法是在 `handlers` 结构体中使用 options.

在这个方法中我们使用一个默认的实现,除非用户提供一个作为选项的额外方式.

`handlers` 构造器成为了一个可变参数的方法, 所以可以给定参数或者不给.

在下面的例子中, 我们导出一个叫做 `WithCustomClient` 的高阶函数, 以便读者能够更加容易的使用可选 `Client` 实现

```golang
package handlers

import "net/http"

type handlers struct {
	client *http.Client
}

type handlersOptions struct {
	client *http.Client
}

func WithCustomClient(client *http.Client) func(*handlersOptions) *handlersOptions {
	return func(ho *handlersOptions) *handlersOptions {
		ho.client = client
		return ho
	}
}

func NewHandlers(options ...func(*handlersOptions) *handlersOptions) *handlers {
	ops := &handlersOptions{
		client: http.DefaultClient,
	}

	for i := range options {
		ops = options[i](ops)
	}

	return &handlers{
		client: ops.client,
	}

}

```



使用构造函数，我们现在可以通过不带任何参数调用 NewHandlers 来使用默认实现 (http.DefaultClient)：



```golang
h := handlers.NewHandlers()
```



或者可以通过以自定义的 `Client` 选项来调用它:
```golang
h := handlers.NewHandlers(handlers.WithCustomClient(&http.Client{Timeout: time.Duration(time.Second * 10),}))
```





使用这种方式,当需要想要使用默认实现时,不需要自己构建这些实现,这可以让你的主文件不至于那么错综复杂.







### Test Implementation


现在我们已经谈论到注入不同实现的方法, 我们接下来将会创建一个实现来作为测试使用.

这里我们会使用高阶函数的方式来注入依赖.



在这一点上，我们可以自己编写一个测试替身，也可以使用 GoMock 生成一个实现，允许我们通过存根控制其行为。


我们使用  GoMock.

通过将下面的代码行添加到我们的代码库中，我们可以从客户端界面自动生成一个测试替身。



... ...
省略生成的代码 



这是我们可以在测试中使用的 Client 实现.
我们可以注入它而不是使用默认的 Http client,并模拟对 Kanye API进行真正地 http Get 调用的行为.

我们现在可以在不进行 HTTP 调用的情况下编写测试并测试当我们得到错误响应时会发生什么.



以下是一个例子,展示我们所能编写的测试场景类型.

测试使用行为驱动开发的测试框架 Ginkgo.


... ...
省略


测试用例存根模拟客户端实现。 我们检查它是否被正确的 URL 字符串调用，并告诉它返回我们的 HTTP 响应而没有错误。 我们可以检查 HTTP 状态代码和发送给用户的报价是否符合我们的预期。



## Conslusion

我们的谈论覆盖了如何使用依赖倒置的来变更我们的 handler 来达到使用不同 Client 实现的目的.
这项技术可以应用到任何类型的依赖中.


依赖倒置可以成为创建程序的强大工具,这些程序随着功能的增加而更加稳定与健壮.

我们可以通过模拟实现来创建测试脚本.

通过依赖概念而不是具体的实现,我们减少了对业务逻辑的更改次数.


想要阅读更多关于依赖倒置的文章,我强烈推荐 Robert C Martin 的 Clean Architecture.
































