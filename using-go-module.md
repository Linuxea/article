
Go 1.11 与 1.12 主要包含了
- 对模块的支持
- 新的依赖管理系统,明确依赖的版本信息与更易于管理

模块是根目录存在 go.mod与一系列 go 包文件的树集合.
go.mod 文件定义了一个模块的模块路径,同时也是根目录使用的导入路径,以及它的依赖需求.

每一项依赖包含模块路径和版本.


在 Go 1.11 中,go 命令对于当前目录或者父目录存在 go.mod 文件,并且目录位于 $GOPATH/src 之外的目录提供了 modules 支持.(为了向后兼容,如果位于 $GOPATH/src 目录内, go 命令依然使用 GOPATH 模式,即使存在 go.mod 文件)
从 Go1.13 开始,module 模式成为所有开发的默认选项


本文按顺序介绍使用 go module 常用的操作
- 创建一个新的 module
- 添加一个依赖
- 升级依赖
- 添加一个
- 升级依赖到新的主要版本
- 删除没有使用的依赖



## Creating a new module

> note export GO111MODULE=off

在 $GOPATH/src 目录外创建一个新的空目录,创建然后一个 hello.go 源文件:

```golang
package hello

func Hello() string {
        return "Hello, world."
}
```

为这个函数同样创建一个测试文件:
```golang
package hello

import "testing"

func TestHello(t *testing.T) {

        want := "Hello, world."
        if got := Hello(); got != want {
                t.Errorf("hello() = %q, want %q", got, want)
        }
}
```


这样的话,目录已经有了一个 package, 不过并不是一个模块,因为没有 go.mod 文件.
我的这个目录路径是: /home/linuxea/Desktop/temp/hhhhh
执行 go test 命令:


Output:
```
PASS
ok      _/home/linuxea/Desktop/temp/hhhhh       0.001s
```

最后一行是包测试的总结.
因为我们工作在非 $GOPATH 目录,同样工作在非模块下,go 命令不知道当前目录的导入路径并且创建一个基于目录名称的假导入路径: _/home/linuxea/Desktop/temp/hhhhh

现在让我们通过 go mod init 来把当前目录变成一个模块根,然后再来 go test:
```bash
➜  hhhhh go mod init example.com/hello
go: creating new go.mod: module example.com/hello
go: to add module requirements and sums:
        go mod tidy
➜  hhhhh go test
PASS
ok      example.com/hello       0.001s
``` 

现在我们完成了第一个模块的创建与测试.

go mod init 命令创建了一个 go.mod 文件:
```bash
➜  hhhhh cat go.mod 
module example.com/hello

go 1.16
```


go.mod 文件只会出现在模块的根目录.
子目录下的包导入形式为模块路径加上子目录的路径.
比如我们创建一个子目录 world, 我们不需要在子目录下面运行 go mod init 命令.
world 包会被自动识别为 example.com/hello 模块的一部分,导入的路径为: example.com/hello/world.


## Adding a dependency

Go modules 的主要动机在于提升使用第三方代码的开发体验.

更新 hello.go 来导入 rsc.io/quote 并且在 Hello 中实现使用:
```golang
package hello

import "rsc.ip/quote"

func Hello() string {
        return quote.Hello()
}
```

重新运行测试:
```golang
➜  hhhhh go test
hello.go:3:8: no required module provides package rsc.ip/quote; to add it:
        go get rsc.ip/quote
```

当遇到导入的包却没有在 go.mod 文件中提供相关依赖时,会提示使用 go get 命令:
```bash
➜  hhhhh go get rsc.io/quote
go get: added rsc.io/quote v1.5.2
```

会把相关依赖的最新版本(最新的定义是最新的稳定版本)添加到 go.mod 文件中, 同时也会下载 rsc.io./quote 使用到的两个依赖 rsc.io/sampler, golang.org/x/test.
但是只有直接依赖会在 go.mod 文件中记录到.
```bash
➜  hhhhh cat go.mod 
module example.com/hello

go 1.16

require rsc.io/quote v1.5.2 // indirect
```

再一次执行 go test 命令时不会重复相同的工作了.
因为 go.mod 已经为最新的编辑结果,同时依赖的模块也已经下载到本地的缓存目录($GOPATH/pkg/mod):
```bash
➜  hhhhh go test
--- FAIL: TestHello (0.00s)
    hello_test.go:9: hello() = "你好，世界。", want "Hello, world."
FAIL
exit status 1
FAIL    example.com/hello       0.001s
```






















