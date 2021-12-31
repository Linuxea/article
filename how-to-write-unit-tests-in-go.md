
# How To Write Unit Tests in Go

> 单元测试是用来测试程序特定代码片断的函数,用来检测应用的准确性,是程序开发语言中至关重要的一环.
在这个章节中,会创建一个小程序并且使用 go 的 testing 包与 go test 命令来构建可以工作的测试套件,包括基于表单的测试,覆盖测试,基准测试与示例文档.

## Prerequisites

需要完成一些前提的准备工作

- 了解 go
- 使用 go 版本1.11及以上

>Note:本案例使用的包管理系统是1.11版本引进的 go modules.Go Modules 用来取代 $GOPATH 并成为go1.13的默认选项.


```bash
go version
go version go1.16.4 linux/amd64
```


## Step 1 - Creating a Sample Program to Unit Test

开始写单元测试之前,需要写用于测试分析的代码.
这一步,我们构建一个小程序,用来计算两个数之和.
在接下来的步骤中,将会使用 go test 来测试这个程序.

首先,创建一个新的目录 math:
```bash
➜  ~ mkdir math
```

进入新创建的目录:
```bash
➜  ~ cd math 
```

这里就是我们的程序根目录.接下来将会在这里执行我们接下来的所有命令.

初始化 Go Modules
```bash
➜  math go mod init testturorial
```

现在,我们使用编辑器创建 math.go 文件:
```bash
➜  math vim math.go
```

math.go 代码如下:
// todo


创建了两个函数 Add 与 Subtract,接收两个整数并返回他们的和或者差.

保存并关闭文件.

这一步完成了 go 编写的代码.接下来开始编写一些测试用例来确保我们的函数能够正常的工作.


## Step 2 - Writing Unit tests in Go

在这一步,我们开始写第一个测试,并且是以 _test.go 命名结尾.
按惯例,go 测试文件与被测试的代码放在同一文件夹或者包中.
当执行 go build 命令的时候,这些测试文件不会被编译器构建,所以不要担心它们会出现在部署文件中.


正如 go 中的所有事物, go 对测试也有新的看法.
Go 语言提供一个小而全的测试包 testing, 开发人员搭配 go test 命令一起使用.
testing 包提供了一些有用的约定,比如覆盖测试与基准测试,这是我们将会探索到的.

使用编辑器创建一个新的文件 math_test.go:
```bash
vim math_test.go
```

一个测试函数的方法签名如下:
```golang
func TestXxxxx(t *testing.T)
```

这意味着所有的测试函数必须以 Test 单词开头, 后面跟着首字母大写的单词后缀.
测试函数只会接收一个参数, testing.T 的指针类型.
这个参数类型包括了输出结果,记录错误,通知失败如 t.Errorf(),等等方法.

在 math_test.go 中添加如下方法:

// todo

首先,在文件中声明了测试包名,导入 testing 包用来使用 testing.T 的方法.测试的代码逻辑放在了 TestAdd 函数.

关于一个测试文件的组成总结一下:
- 有且仅有一个函数参数 t *testing.T
- 测试函数必须以 Test 单词结尾并且跟随首字母大写的单词(按惯例是加上想要测试的函数如:TestAdd)
- 调用方法 t.Error 或者 t.Fail 来表明测试的失败.(t.Error 相比 t.Fail 会更加详细)
- 使用 t.Log 可以提供记录非失败的调试信息记录
- 测试函数们保存文件名惯例为:foo_test.go, 比如这里为 math_test.go


保存并关闭 math_test.go 文件.

在这一步,我们完成了编写第一个测试文件.
下一步,我们会使用 go test 来测试代码.



## Step 3 - Testing Your Go Code Using the go test command

在这一步,我们会使用 go test 强大的命令来完成自动化测试.
go test 可接收不同的标识 flags 来配置想要运行的测试,以及测试返回的输出等等.

在项目的根目录,运行第一个测试:
```bash
➜  math go test
```

你会收到下面的输出:
```
PASS
ok      testturorial    0.001s
```

PASS 意味着代码如期运行.如果是失败的,你会看到 FAIL.

go test 命令只会寻找那些以 _test.go 结尾的文件,然后扫描这些文件中包含 func TestXxxxx(*testing.T) 和后面会提到的其他几个特殊方法, 
go test 会生成一个临时 main 包然后调用,构建与运行,生成结果报告,最后清理.

go test 对于我们的小程序来讲可能够用了,有些时候可能需要查询在跑的测试与各自花费的时间,添加 -v 标识来增加详细程序.

使用新标记来重新运行测试:
```bash
➜  math go test -v
```

输出结果如下:
```bash
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
PASS
ok      testturorial    0.001s
```


在这一步使用了 go test 运行了基础的单元测试.
下一步,我们会写一个复杂点的基于表格驱动的单元测试.





## Step 4 - Writing Table-Driven Tests in Go

基于表格驱动测试与基础单元测试类似,除了它维护着值与结果的表格.
测试套件会迭代表格数据来对代码进行循环测试.
使用这种方法,我们可以测试输入及其各自输出的多种组合.


重新打开 math_test.go 文件:
```bash
➜  math vim math_test.go 
```

删除原来的代码并进行以下代码的替换:

// todo

这里定义了一个结构体以及结构体的列表变量,用数据填充了这个列表,并在测试代码中循环迭代数据用来测试函数.

使用 -v 标记重新运行测试:
```bash
go test -v
```

看到了同样的输出:
```bash
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
PASS
ok      testturorial    0.001s
```

在这一步,我们实现了基于表单的测试.
下一步我们会改为编写覆盖测试.




## Step 5 - Writing Coverage Tests in Go

在这一步我们来编写覆盖测试.

当写测试用例时,一个很重要的关注点是代码的测试覆盖率.上面的例子我们没有给函数 Subtract 编写编写 - 这样我们就能在这时查看一份不完整的测试覆盖.

运行如下命令来计算当前的代码的测试覆盖率:
```bash
➜  math go test -coverprofile=coverage.out
```

输出结果如下:
```bash
PASS
coverage: 50.0% of statements
ok      testturorial    0.001s
```

go 将测试覆盖数据保存在指定的文件 coverage.out 中,我们可以在浏览器中进行查看.
运行如下命令:

```bash
go tool cover -html=coverage.out
```

将会打开默认浏览器,并且渲染出文件只在的覆盖数据:

// todo

绿色的文字部分标识着覆盖,而红色则相反.

在这一步,我们测试了表格驱动的覆盖率.
下一步,我们会对函数进行基准测试.


## Step 6 - Writing Benchmarks in Go

在这一步,我们会编写基准测试.
基准测试用来评估程序的性能,这可能让我们得以评估不同代码实现的以及它们所带来的影响.
利用好基准测试信息,可以揭开源码中值得优化的代码头纱.

在 go 中,以 func BenchmarkXxxx(*testing.B) 形式的代码会被认为是基准测试函数.
go test 会在你提供 -bench 标记时顺序执行这些基准测试.

让我们给单元测试添加一个基准测试吧:

重新打开 math_test.go:
```bash
➜  math vim math_test.go 
```

// todo

基准函数必须运行目标代码 b.N 次,其中 N 是一个可以调整的整数.
在基准执行期间,会调整 b.N. 直到基准函数持续足够长的时间来进行可靠的计时.
-bench 标志接收正则形式的参数.

保存并保存文件.
再次使用 go test 来运行基准测试.

```bash
➜  math go test -bench=.
```

. 会匹配文件中每个基准测试函数.或者你也可以指定特定的基准测试函数:

```bash
➜  math go test -bench=Add
```

上面的命令执行结果如下:
```bash
goos: linux
goarch: amd64
pkg: testturorial
cpu: Intel(R) Core(TM) i5-9400 CPU @ 2.90GHz
BenchmarkAdd-6          1000000000               0.2550 ns/op
PASS
ok      testturorial    0.284s
```

结果表明以每次循环时间为 0.2550ns 执行了 1,000,000,000 次.

>不要在系统繁忙时进行基准测试,否则会对基准测试进行干扰从而得到错误的结果.


在这一步,我们添加了基准测试来增长我们的单元测试.
在下一步也是最后一个步骤,我们会给文档添加示例, go test 同样会对此进行验证.

## Step 7 -- Documenting Your Go Code with Examples

在这个步骤,我们会用例子来给代码做文档,并进行例子的测试.
Go 非常注重正确的文档, 示例代码为文档和测试添加新的维度.
示例是基于真实存在的方法和函数.你的例子应该向用户展示如何使用特定的代码片段.
Example 函数是第三种会被 go test 特殊对待的函数.

重新打开 math_test.go 文件:

```bash
➜  math vim math_test.go
```

在结尾添加新的示例函数:

// todo


Output: 这一行指定与记录了期望的输出.

>注意：比较忽略前导和尾随空格.

保存并关闭文件.

重新运行单元测试:

```bash
➜  math go test -v
```

最新的输出如下:
```bash
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
=== RUN   ExampleAdd
--- PASS: ExampleAdd (0.00s)
PASS
ok      testturorial    0.001s
```

你的示例代码此时也被测试了.这个特性可能提高我们文档与单元测试的健壮性.


## Conclusion

在这个章节中,我们创建了一个小型程序并随便为它编写基础测试,基于表格的测试,覆盖测试与示例文档.


作为一名程序员,花费时间来编写足够的单元测试是非常有用的.这提高了你对代码或程序的持续按预期运行的信心.
testing 包提供了可观的单元测试功能.如果想要学习更多,可以查看 go 的官方文档.


## 参考

- [1] [How To Write Unit Tests in Go](https://www.digitalocean.com/community/tutorials/how-to-write-unit-tests-in-go-using-go-test-and-the-testing-package)
- [2] [testing package](https://pkg.go.dev/testing)