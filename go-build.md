# Customing Go Binaries with Build Tags


## Introduction

Go 的构建标签决定文件是否引入构建过程。

这个功能可以使开发者以一种快速且有组织的方式从是同一份源码中构建不同的应用版本。

许多开发者使用构建标签来提升跨平台兼容应用构建的工作流。
比如变更代码来解决不同的操作系统间的差异。


本方以一个应用示例来演示如何使用构建标签，此应用示例提供了 Free, Pro 与 Enterprise 不同级别的功能集合，在面对不同的客户时提供不同的应用版本。



## Prepare & Free Version

```zsh
➜  gobuildsample pwd
/mnt/c/Users/Linux/Desktop/code/gobuildsample
➜  gobuildsample go mod init linuxea.com/app
go: creating new go.mod: module linuxea.com/app
➜  gobuildsample touch feature.go
➜  gobuildsample touch free.go
➜  gobuildsample touch main.go
```

feature.go
```go
package main

var features []string
```


free.go
```go
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


代码比较简单。
`feature.go` 文件声明了一个功能集合切片， `free.go` 通过 `init()` 来注入免费的功能特性，免费特性同时也是默认特性，不需要额外指定就能够存在于最终构建文件中，`main.go` 文件则是打印出版本的所有功能集。

```zsh
➜  gobuildsample git:(master) go build -o app .
➜  gobuildsample git:(master) ✗ ./app
freeFeature#1
freeFeature#2
```



## Pro Version

```zsh
➜  gobuildsample touch pro.go
```

pro.go
```go
// +build pro

package main

func init() {
	features = append(features, "proFeature#1")
}
```

注意到 `pro.go` 文件首行是构建标签的声明，意味着构建过程中如果没有 pro 标签则不会引入此文件。

```zsh
➜  gobuildsample git:(master) ✗ go build -o app -tags pro .
➜  gobuildsample git:(master) ✗ ./app
freeFeature#1
freeFeature#2
proFeature#1
➜  gobuildsample git:(master) ✗ go build -o app .
➜  gobuildsample git:(master) ✗ ./app
freeFeature#1
freeFeature#2
```

当两个版本时一切工作顺利，但是事情在添加第三种标签时变得更加复杂了点。


## Enterprise Version

```zsh
➜  gobuildsample touch enterprise.go
```

enterprise.go
```go
// +build enter

package main

func init() {
        features = append(features, "enterpriseFeature#1", "enterpriseFeature#2")
}
```

还是同样的套路, 声明所需要的 `pro` 标签。

```zsh
➜  gobuildsample git:(master) ✗ go build -o app -tags 'pro enter' .
➜  gobuildsample git:(master) ✗ ./app
enterpriseFeature#1
enterpriseFeature#2
freeFeature#1
freeFeature#2
proFeature#1
```

事情依然很顺利。
但是我们在添加构建标签时 `enter` 时，开始变得有些微妙。

```zsh
➜  gobuildsample git:(master) ✗ go build -o app -tags enter .
➜  gobuildsample git:(master) ✗ ./app
enterpriseFeature#1
enterpriseFeature#2
freeFeature#1
freeFeature#2
```

这在逻辑上并不行得通。当我们拥有 enter 版本时却无法拥有 pro 版本的特性，然而却能构建成功，这是不合理的。



## Build Tag Boolean Logic

当我们向文件中添加多个标签时，标签之间会进行逻辑运算。

让我们重新编辑下 enterprise.go:

```go
➜  gobuildsample git:(master) ✗ cat enterprise.go
// +build enter,pro

package main

func init() {
        features = append(features, "enterpriseFeature#1", "enterpriseFeature#2")
}
```

这意味着标签之间存在的是逻辑与的关系。二者需要同时指定。

```zsh
➜  gobuildsample git:(master) ✗ go build -o app -tags enter .
➜  gobuildsample git:(master) ✗ ./app
freeFeature#1
freeFeature#2
➜  gobuildsample git:(master) ✗ go build -o app -tags 'pro enter' .
➜  gobuildsample git:(master) ✗ ./app
enterpriseFeature#1
enterpriseFeature#2
freeFeature#1
freeFeature#2
proFeature#1
```

构建时指定所需要的全部标签时，`enterprise.go`  文件才会被引入到构建过程中。这看起来逻辑上更加合理。

除了上面提到的逻辑与关系，还存在着其他的逻辑运算关系：

Build Tag Syntax	Build Tag Sample	Boolean Statement
Space-separated elements	// +build pro enterprise	pro OR enterprise
Comma-separated elements	// +build pro,enterprise	pro AND enterprise
Exclamation point elements	// +build !pro	NOT pro

这让构建场景变得更加灵活。


## fsnotify

`fsnotify` 是一款 Go 跨平台文件系统通知工具。

利用它可以打造出简单的配置文件实时更新服务。

能够在不同操作系统上监听文件的变更，其在内部就使用了构建标签来解决不同操作系统所带来的差异。


## Conclusion

In this tutorial, you used build tags to allow you to control which of your code got compiled into the binary. First, you declared build tags and used them with go build, then you combined multiple tags with Boolean logic. You then built a program that represented the different feature sets of a Free, Pro, and Enterprise version, showing the powerful level of control that build tags can give you over your project.

在本文例子中，我们使用了 `build tag` 控制哪些文件引入到构建过程。
从声明标签到在 go build 中指定标签，然后是以布尔运算来结合运算多个标签（与是非）。
构建标签可以为您提供对项目的强大控制级别。


## 参考

- [1] [Customizing Go Binaries with Build Tags](https://www.digitalocean.com/community/tutorials/customizing-go-binaries-with-build-tags)
- [2] [fsnotify](https://github.com/fsnotify/fsnotify)