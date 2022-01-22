# Stop debugging Go with Println and use Delve instead


## Foread

从 "Hello, world!"开始学习一门语言,然后开始编写一些简单的示例代码并执行, 遇到问题时进行细微的修复然后继续前行.
我相信无论使用什么技术都是拥有相同的经历.

但是,如果你确实想要使用一种语言一段时间,并期望能够对它足够熟悉,有好些事情能够在这条道路上帮助你.


其中一件就是 debugger.

有些人可能偏爱在代码中使用简单的 "print" 语句来做调试,这在很少行数的程序中是可行的.

但是,如果你工作在一个大型项目中,并且与多位开发者和成千上万行代码打交道时, 向 debugger 上的投资是有意义的.

最近我开始学习 go 编程语言, 在这篇文章, 我们一起来探索一款叫做 Delve 的调试器 (debugger).

Delve 是一款为 go 量身定制的特殊工具,我们会通过一些示例代码来探讨其特性, 不过你也不需要担心文章中出现的 go 程序,
因为它们是易于理解的,即使你从未使用 go 编程. 

简单是 go 语言的目标之一, 所以这里的代码也是一致的,容易理解与解释.



## Introduction to Delve

Delve is a debugger for the Go programming language. The goal of the project is to provide a simple, full featured debugging tool for Go. Delve should be easy to invoke and easy to use. Chances are if you're using a debugger, things aren't going your way. With that in mind, Delve should stay out of your way as much as possible.



### Golang Installation
```bash
➜  ~ go version
go version go1.17.2 linux/amd64
➜  ~ echo $GOROOT
/home/linuxea/soft/go
➜  ~ echo $GOBIN
/home/linuxea/soft/go/bin

```



### Delve Installation
```bash
$ go install github.com/go-delve/delve/cmd/dlv@latest
```

```bash
➜  ~ dlv version
Delve Debugger
Version: 1.7.3
Build: $Id: 4c81a2a322716591167f0bd690f7ef656781b6f4 $
```





现在, 让我们使用结合 Delve 与 Go 程序来理解其特性并且知道如何使用它.

我们先创建一个 Go 项目, 并以展示 "Hello, world!" 信息的 Go 程序开始.


// todo 图片


## Loading a program in Delve

有两种方式将程序装载到 Delve Debugger.

### Using the debug argument when source code is not yet compiled to binary.

第一种方式是在只需要源文件时使用 debug 命令.
Delve 编译程序成以 __debug_bin 为名称的二进制文件, 然后将其加载到调试器当中.


在这个例子中, 在 hello.go 所在目录下运行 dlv debug 命令.

如果目录下有多个源文件并且各自存在 main 函数时, Delve 可能会抛出 error, Delve 的预期是从单个程序或者项目中构建二进制文件.

如果这种情况出现, 你最好使用稍后会提到的第二种方式来调试.

现在我们继续第一种


todo 图片

如上图, 在列出的目录文件中,我们可以看到额外的 __debug_bin 二进制文件, 它是从源码中编译过来并且加载到了调试器当中.





### Using the exec argument

当您拥有预编译的 Go 二进制文件，或者您已经使用 go build 命令编译的二进制文件，并且不希望 Delve 将其编译为 `__debug_bin` 命名的二进制文件时，将程序加载到 Delve 的第二种方法非常有用。

这种情况下,可以使用 exec 命令来将程序加载到调试器.



## Getting help within Delve












