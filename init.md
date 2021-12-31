## init


```golang
package main

func main() {

	initMysql()

	initRedis()

	initElasticSearch()

	initRocketMQ()
	
	// init others

}

```

是否觉得这样的手动初始化依赖的第三方组件过于丑陋?

按照依赖的传递性与分层结构,这不应该也不能是顶层所应该关心的事.



## 前言

当使用进行应用开发时,有些时候可能需要在项目启动的时候做一些配置上的初始化, 比如创建数据库的连接,或者加载本地文件等等.

当使用 <code>go</code> 开发时,这个时候就轮到 <code>init</code> 函数上场了.



## init 函数

在 <code>go</code> 中,<code>init</code> 函数十分强大, 相较于其他的编程语言所相似的特性,<code>init</code> 要更加容易使用得多.

<code>init</code> 函数在包块中使用,不管当前包被其他包导入多少次, <code>init</code> 函数只会执行一次.


执行一次这个特性是时候引起我们的注意. 这可以允许我们有效地配置数据库连接,注册服务中心,或者执行一系列只会做一遍的任务.



```golang
package main

import "fmt"

func main() {
	fmt.Println("我是 main")
}

func init() {
	fmt.Println("我是init, 我会跑在 main 前面")
}

```


注意到上面的例子,我们没有在程序的任何地方显示地调用 <code>init</code> 函数.

<code>go</code> 为我们隐匿地执行了处理.

输出结果应该为:
```bash
我是init, 我会跑在 main 前面
我是 main
```


这很6.

我们开始做些更加 cool 的事情吧.

```golang
package main

import "fmt"

var name string

func main() {
	content := fmt.Sprintf("我的名字是%s", name)
	fmt.Println(content)
}

func init() {
	name = "焦迈奇"
}

```

在上面这个例子中, 可以开始发现为什么相比显式调用设置的调用, 使用 <code>init</code> 函数会更加的受欢迎与偏好.这让我们更加能更好分离关注点-配置与使用.

当运行上面的例子, 变量 <code>name</code> 已经被正确的设置了.


## 多包下情况

我们来看一下更加贴近生产环境的案例.

想像一下我们有3个不同的包在我们的应用程序中,<code>main</code>, <code>broker</code> 与 <code>database</code> 或者其他.

在每个包中,我们可以指定 <code>init</code> 函数来执行一些数据库连接到第三方服务(比如 mysql, kafka)等任务.

不管我们调用多少次,在使用数据库资源时, 使用的都是在 <code>init</code> 函数中设置好的数据库连接.


> note: 有一点至关重要,不要依赖于 init 函数的执行顺序.而是要设计出无关 init 执行顺序的程序系统.


## 初始化的顺序

对于更复杂的系统来说,在给定的一个包下,可能存在多个代码文件. 

每个文件都拥有自己的 <code>init</code> 函数.

所以这种情况下, <code>go</code> 会怎样的组织他们的执行顺序呢?


当谈到初始化的顺序时，需要考虑一些事情。 

Go 中的事物通常按声明顺序的顺序初始化，或者在它们可能依赖的任何变量之后初始化。 

这意味着，如果你在同一个包中有 2 个文件 <code>a.go</code> 和 <code>b.go</code>，如果 <code>a.go</code> 中任何东西的初始化依赖于 <code>b.go</code> 中的东西，
它们将首先被初始化。

举个粟子

service 包中有两个文件 
- user
- book


user
```golang
package service

import "fmt"

var bookService = NewBookService()

type UserService struct {
}

func (s UserService) Borrow(bookName string) {
	fmt.Println(fmt.Sprintf("教练,我想要借本书%s", bookName))
	bookService.Record(bookName)
}

func init() {
	fmt.Println("user service init")
}

```


book
```golang
package service

import "fmt"

type BookService struct {
}

func (s BookService) Record(bookName string) {
	fmt.Println("记录下来..." + bookName)
}

func init() {
	fmt.Println("book service init")
}

func NewBookService() BookService {
	return BookService{}
}

```

准备 main 文件 
```golang
package main

import "demo/service"

func main() {

	userService := service.UserService{}
	userService.Borrow("本草纲目")

}
```

因为 <code>user</code> 文件的变量 <code>bookService</code> 依赖于 <code>book</code> 文件, 所以变量 <code>bookService</code> 将会被首先初始化.

等到所有依赖的变量完成初始化,接着才是按顺序决定 <code>init</code> 函数的执行.



## 同个文件下的多个 init 函数

如果有同个文件出现多个 <code>init</code> 函数会发生什么?

一开始我也不相信这是真的.

但是 <code>go</code> 确实提供这个能力.

这些 <code>init</code> 函数在文件中按照各自的声明顺序被调用.

```golang
package main

import "fmt"

func main() {
	// empty
}

func init(){
	fmt.Println("init one")
}

func init(){
	fmt.Println("init two")
}

func init(){
	fmt.Println("init three")
}
```


输出:
```bash
init one
init two
init three
```

## 其他

除了变量的依赖关系使得 <code>init</code> 函数首先执行变得不再是那么确定.
除此之外, 常量的初始化会比变量更加高级.

这里给出一张生动的图来简单了解:




## 总结

本文包含了 init 世界里基础的介绍. 一旦你掌握了包初始化的使用,你可能会发现对你组织项目基础结构是非常有帮助的.




## 参考

- [1] [The Go init Function](https://tutorialedge.net/golang/the-go-init-function/)
- [2] [When is the init() function run?](https://stackoverflow.com/questions/24790175/when-is-the-init-function-run)