# Customing Go Binaries with Build Tags


## Introduction

In Go, a build tag, or a build constraint, is an identifier added to a piece of code that determines when the file should be included in a package during the build process. This allows you to build different versions of your Go application from the same source code and to toggle between them in a fast and organized manner. Many developers use build tags to improve the workflow of building cross-platform compatible applications, such as programs that require code changes to account for variances between different operating systems. Build tags are also used for integration testing, allowing you to quickly switch between the integrated code and the code with a mock service or stub, and for differing levels of feature sets within an application.


Go 的构建标签决定文件是否引入构建过程。

这个功能可以使开发者从是同一份源码中构建不同的应用版本，并且是以一种快速且有组织的方式。

许多开发者使用构建标签来提升跨平台兼容应用构建的工作流。
比如变更代码来解决不同的操作系统间的差异。

Let’s take the problem of differing customer feature sets as an example. When writing some applications, you may want to control which features to include in the binary, such as an application that offers Free, Pro, and Enterprise levels. As the customer increases their subscription level in these applications, more features become unlocked and available. To solve this problem, you could maintain separate projects and try to keep them in sync with each other through the use of import statements. While this approach would work, over time it would become tedious and error prone. An alternative approach would be to use build tags.

本方我们以一个不同客户功能集合作为例子。

在写一些应用程序时，你可能想要控制哪一些功能来引入到二进制可执行文件中，比如一个应用提供了 Free, Pro 与 Enterprise 不同级别，随着用户提升订阅等级，就能够解锁更多的功能使其可用。

为了解决这个问题，你可以维护不同的项目并且通过导入语句来保持它们之间的同步，这虽然可行，然而随着时间推移会变得冗余与易错。

另外一个可行的方案是使用构建标签。

In this article, you will use build tags in Go to generate different executable binaries that offer Free, Pro, and Enterprise feature sets of a sample application. Each will have a different set of features available, with the Free version being the default.


在本方中，你将会在 Go 中使用构建标签来生成不同的可执行二进制文件，分别提供了 Free, Pro 与 Enterprise 功能集合，Free 是默认的。




## Building the Free Version

Let’s start by building the Free version of the application, as it will be the default when running go build without any build tags. Later on, we will use build tags to selectively add other parts to our program.

我们通过构建 Free 版本的应用作为开始，同时也是运行 `go build` 时没有指命任意构建标签的默认版本。

```golang
➜  gobuildsample pwd
/mnt/c/Users/Linux/Desktop/code/gobuildsample
➜  gobuildsample go mod init linuxea.com/gobuildsample
go: creating new go.mod: module linuxea.com/gobuildsample
➜  gobuildsample touch main.go
➜  gobuildsample touch feature.go
➜  gobuildsample touch free.go
```

feature.go
```go
package main

var features []string
```

free.go
```golang
package main

func init() {
	features = append(features, "freeFeature#1", "freeFeature#2")
}
```


main.go
```go
package main

import "fmt"

func main() {

	for _, v := range features {
		fmt.Println(v)
	}
}
```

In this file, we created a program that declares a slice named features, which holds two strings that represent the features of our Free application. The main() function in the application uses a for loop to range through the features slice and print all of the features available to the screen.

上述程序中，我们创建变量为切片类型的 features.
`free.go` 使用 `init` 追加了两个字符串来代表免费的功能特性, `main.go` 通过循环打印出所有可用功能。


Save and exit the file. Now that this file is saved, we will no longer have to edit it for the rest of the article. Instead we will use build tags to change the features of the binaries we will build from it.

保存并退出文件，在剩下的时间我们不会再编辑这些基础文件，而是通过构建标签方式来改变二进制文件中的功能集合。

```zsh
➜  gobuildsample git:(master) ✗ go build -o main.go .
➜  gobuildsample git:(master) ✗ ./main.go
freeFeature#1
freeFeature#2
```


## Adding the Pro Features 

We have so far avoided making changes to main.go, simulating a common production environment in which code needs to be added without changing and possibly breaking the main code. Since we can’t edit the main.go file, we’ll need to use another mechanism for injecting more features into the features slice using build tags.

Let’s create a new file called pro.go that will use an init() function to append more features to the features slice:

在不改变原来文件的前提下，通过其他方式注入更多功能。

我们创建一个新文件 `pro.go` 与 `init()` 函数来追加功能到切片中。

```zsh
➜  gobuildsample touch pro.go
```


pro.go
```golang
package main

func init() {
	features = append(features, "proFeature#1")
}
```


```zsh
➜  gobuildsample git:(master) ✗ go build -o main .
➜  gobuildsample git:(master) ✗ ./main
freeFeature#1
freeFeature#2
proFeature#1
```

The application now includes both the Pro and the Free features. However, this is not desirable: since there is no distinction between versions, the Free version now includes the features that are supposed to be only available in the Pro version. To fix this, you could include more code to manage the different tiers of the application, or you could use build tags to tell the Go tool chain which .go files to build and which to ignore. Let’s add build tags in the next step.

应用现在包含了 Pro 与 Free 两种功能。然而这还不是我们想要的：这两种版本没有区别，Free 版本拥有 Pro 才能提供的功能。
为了进行修复，接下来使用构建标签来告诉 Go 工具链哪些 Go 文件需要构建而哪一些需要忽略。


## Adding Build Tags

You can now use build tags to distinguish the Pro version of your application from the Free version.

Let’s start by examining what a build tag looks like:

我们先从查看构建标签是长怎样的开始：

```go
// +build tag_name
```

By putting this line of code as the first line of your package and replacing tag_name with the name of your build tag, you will tag this package as code that can be selectively included in the final binary. Let’s see this in action by adding a build tag to the pro.go file to tell the go build command to ignore it unless the tag is specified. Open up the file in your text editor:

将这一行放到文件的第一行并将 tag_name 替换为你的构建标签，
