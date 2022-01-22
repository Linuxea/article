# Golang Debugging with Delve - [Step by Step]



> 在新手学习一门新语言的时候,或者梳理复杂的代码逻辑时, debugger 即调试器往往扮演着梳理脉络的重要角色.


这篇文章中,我们将会学习如何使用 `Delve` 来调试 Go 程序.

`Delve` 是一款 Go 的第三方调试器, 在 [https://github.com/go-delve/delve](https://github.com/go-delve/delve) 上是可获取的.

它是相比于 `GDB` 之外的另外一款可供选择的 GO 程序调试器, 但相较 `GDB`, 它对于 GO 的调试拥有更多丰富的特性.


>需要注意的是,当调用使用标准工具链构建的 Go 程序时, `Delve` 是相比于 `GDB` 更好的替代方案.
它更能理解 Go 的运行时, 数据结构与表达式. `Delve` 目前支持 amd64 架构上的 Linux, OSX, 和 Windows.关于最新的支持列表清单,可以查看 [the Delve documentation](https://www.baidu.com)



## Objective

学习文章到最后, 你可以学会轻易地使用 `Delve` 命令行工具来调试与检查 Go 程序.

我们会学习如何在程序中查看, 添加或者改变断点,通过按行或者断点来导航程序,检查变量,函数与表达式的值

最终详尽地来分析 Go 程序.


## Download and Install Go Delve

以下命令可以在 Linux, macOS, Windows 与 FreeBSD 上面被识别

先检查相关环境变量是否正确设置
```
➜  ~ echo $GOROOT
/home/linuxea/soft/go
➜  ~ echo $GOBIN
/home/linuxea/soft/go/bin
```

Go 1.6 或者更新的版本,可以使用如下命令来安装

```
$ go install github.com/go-delve/delve/cmd/dlv@latest
```


你可以通过查看 `go help install` 来查看可执行文件 `dlv` 被安装到哪里.



## Check Your Installation

检查安装是否成功
```
➜  ~ dlv version
Delve Debugger
Version: 1.7.3
Build: $Id: 4c81a2a322716591167f0bd690f7ef656781b6f4 $
```
上述结果表示你已经安装成功!

## Start Debugging (Delve Server)

当我们提示开始调试时, 指的是开启一个"调试会话".

你可以通过 `dlv help` 命令来查看如何通过哪些命令之一来开启一个调试会话.


```
➜  ~ dlv help
...

Usage:
  dlv [command]

Available Commands:
  attach      Attach to running process and begin debugging.
  connect     Connect to a headless debug server.
  core        Examine a core dump.
  dap         [EXPERIMENTAL] Starts a headless TCP server communicating via Debug Adaptor Protocol (DAP).
  debug       Compile and begin debugging main package in current directory, or the package specified.
  exec        Execute a precompiled binary, and begin a debug session.
  help        Help about any command
  run         Deprecated command. Use 'debug' instead.
  test        Compile test binary and begin debugging program.
  trace       Compile and begin tracing program.
  version     Prints version.

...
```

上面的命令中使我们感兴趣的有 `dlv debug` 与 `dlv exec`.

这两个命令都是用来开启一个调试会话,唯一的区别是 `dlv debug` 需要从源码编译出二进制文件, 而 `dlv exec` 预期的是一个编译好的二进制文件.

> dlv test 命令在我们需要调试 go 单元测试时同样有用


## Golang Debugging Code Example


如下表示的是我们将用于调试的代码片段. 它是一个简单的求和实现.

```golang
package main

import "fmt"

func main() {

	first := 1
	second := 9

	result := add(first, second)

	fmt.Println(result)

}

func add(a, b int) int {
	return a + b
}
```

我们需要传递一个将被编译与执行以进行调试的 main 包.

```
➜  dlv debug main.go 
Type 'help' for list of commands.
(dlv) 
```

这开启了一个调试会话, 也被称为 `delve` 服务, 因为它是一个正在等待指令的运行中进程.


如前面提到的, 也可以使用 `dlv exec` 来开启一个调试会话.它需要一个编译好的二进制文件.
```
➜  go build -o __debug_bin main.go
➜  dlv exec ./__debug_bin
Type 'help' for list of commands.
(dlv) 
```




## The Delve Client

一旦调试会话开启,我们已经编译好并附着到 GO 二进制文件中,以便开始调试工作.

我们现在看到的是一个新的 `repl` 界面, 它是 delve 的解释器, 也可以叫做 delve 客户端, 它会把调试指令发送给我们上一步创建好的 delve 服务端.

我们可以输入以下命令来查看所有可用的命令
```
Type 'help' for list of commands.
(dlv) help
The following commands are available:

Running the program:
    call ------------------------ Resumes process, injecting a function call (EXPERIMENTAL!!!)
    continue (alias: c) --------- Run until breakpoint or program termination.
    next (alias: n) ------------- Step over to next source line.
    rebuild --------------------- Rebuild the target executable and restarts it. It does not work if the executable was not built by delve.
    restart (alias: r) ---------- Restart process.
    step (alias: s) ------------- Single step through program.
    step-instruction (alias: si)  Single step a single cpu instruction.
    stepout (alias: so) --------- Step out of the current function.

Manipulating breakpoints:
    break (alias: b) ------- Sets a breakpoint.
    breakpoints (alias: bp)  Print out info for active breakpoints.
    clear ------------------ Deletes breakpoint.
    clearall --------------- Deletes multiple breakpoints.
    condition (alias: cond)  Set breakpoint condition.
    on --------------------- Executes a command when a breakpoint is hit.
    toggle ----------------- Toggles on or off a breakpoint.
    trace (alias: t) ------- Set tracepoint.
    watch ------------------ Set watchpoint.

Viewing program variables and memory:
    args ----------------- Print function arguments.
    display -------------- Print value of an expression every time the program stops.
    examinemem (alias: x)  Examine raw memory at the given address.
    locals --------------- Print local variables.
    print (alias: p) ----- Evaluate an expression.
    regs ----------------- Print contents of CPU registers.
    set ------------------ Changes the value of a variable.
    vars ----------------- Print package variables.
    whatis --------------- Prints type of an expression.

Listing and switching between threads and goroutines:
    goroutine (alias: gr) -- Shows or changes current goroutine
    goroutines (alias: grs)  List program goroutines.
    thread (alias: tr) ----- Switch to the specified thread.
    threads ---------------- Print out info for every traced thread.

Viewing the call stack and selecting frames:
    deferred --------- Executes command in the context of a deferred call.
    down ------------- Move the current frame down.
    frame ------------ Set the current frame, or execute command on a different frame.
    stack (alias: bt)  Print stack trace.
    up --------------- Move the current frame up.

Other commands:
    config --------------------- Changes configuration parameters.
    disassemble (alias: disass)  Disassembler.
    dump ----------------------- Creates a core dump from the current process state
    edit (alias: ed) ----------- Open where you are in $DELVE_EDITOR or $EDITOR
    exit (alias: quit | q) ----- Exit the debugger.
    funcs ---------------------- Print list of functions.
    help (alias: h) ------------ Prints the help message.
    libraries ------------------ List loaded dynamic libraries
    list (alias: ls | l) ------- Show source code.
    source --------------------- Executes a file containing a list of delve commands
    sources -------------------- Print list of source files.
    types ---------------------- Print list of types
```



虽然有不少的命令,但是我们可以根据专注领域的不同来划分, 以便更好地理解 delve 客户端.




## General Delve commands

### list

第一个吸引读者注意力的是 `list` 命令, 它可以列出给定位置的源码.

我们可以通过一个包名和函数名称,或者一个文件路径组合行数来作为参数.


#### shows source code by package and function name

通过包名与函数名称来展示代码 

```
(dlv) list main.main
Showing /home/linuxea/Desktop/temp/main.go:9 (PC: 0x4947ea)
     4:
     5: func add(a, b int) int {
     6:         return a + b
     7: }
     8:
     9: func main() {
    10:
    11:         first := 1
    12:         second := 9
    13:
    14:         result := add(first, second)
```

#### shows source code by file name and line number

通过文件名与行号来展示代码

```
(dlv) list main.go:5
Showing /home/linuxea/Desktop/temp/main.go:5 (PC: 0x4947a0)
     1: package main
     2:
     3: import "fmt"
     4:
     5: func add(a, b int) int {
     6:         return a + b
     7: }
     8:
     9: func main() {
    10:
```


### Funcs

通过给定的正则表达式来寻找函数
```
(dlv) funcs add
...
fmt.(*fmt).writePadding
internal/reflectlite.add
main.add
...
```


### Exit

当你觉得自己困在会话中时, 可以使用 `exit` 退出会话



## Adding Breakpoints with Delve

一旦你知道了如何通过 `list` 命令在屏幕上展示代码片段, 你应该想开始在程序中添加断点, 断点指的是你想要程序运行时停止的地方, 从而可以检查代码中的变量值与表达示值.

在上面的示例文件 `main.go` 中, 我们此处简单演示在如何在第 11 行添加一个断点, 可以通过 `break` 关键字加上我们想要添加断点的位置.


### break

这将在指定的位置添加一个断点,并列出该断点的使用位置

```
(dlv) break main.go:11
Breakpoint 1 set at 0x4947f8 for main.main() ./main.go:11
(dlv) list main.go:11
Showing /home/linuxea/Desktop/temp/main.go:11 (PC: 0x4947f8)
     6:         return a + b
     7: }
     8:
     9: func main() {
    10:
    11:         first := 1
    12:         second := 9
    13:
    14:         result := add(first, second)
    15:
    16:         fmt.Println(result)
```


### breakpoints

列出当前调试会话中的所有断点

```bash
(dlv) breakpoints
Breakpoint runtime-fatal-throw (enabled) at 0x432ca0 for runtime.throw() /home/linuxea/soft/go/src/runtime/panic.go:1188 (0)
Breakpoint unrecovered-panic (enabled) at 0x433000 for runtime.fatalpanic() /home/linuxea/soft/go/src/runtime/panic.go:1271 (0)
        print runtime.curg._panic.arg
Breakpoint 1 (enabled) at 0x4947f8 for main.main() ./main.go:11 (0)
```

在 `breakpoints` 结果中我们可以看到 3 个断点信息.
前 2 个是由 `delve` 自动添加的, 用作出现 `panic` 和 `error` 的保护措施，以便我们能够确定程序的状态并检查变量、堆栈跟踪和状态。

第三个断点表明 `Breakpoint 1` 是我们刚刚在第 11 行添加的那一个.



### clear

移除会话中指定的断点

```bash
(dlv) clear 1
Breakpoint 1 cleared at 0x4947f8 for main.main() ./main.go:11
```

在错误地添加断点或者当你想要调试其他区域的代码的时候是有用的


### clearall

删除所有由我们手动添加的断点

```
(dlv) break main.go:12
Breakpoint 2 set at 0x494801 for main.main() ./main.go:12
(dlv) break main.go:14
Breakpoint 3 set at 0x49480a for main.main() ./main.go:14
(dlv) breakpoints
Breakpoint runtime-fatal-throw (enabled) at 0x432ca0 for runtime.throw() /home/linuxea/soft/go/src/runtime/panic.go:1188 (0)
Breakpoint unrecovered-panic (enabled) at 0x433000 for runtime.fatalpanic() /home/linuxea/soft/go/src/runtime/panic.go:1271 (0)
        print runtime.curg._panic.arg
Breakpoint 2 (enabled) at 0x494801 for main.main() ./main.go:12 (0)
Breakpoint 3 (enabled) at 0x49480a for main.main() ./main.go:14 (0)
(dlv) clearall
Breakpoint 2 cleared at 0x494801 for main.main() ./main.go:12
Breakpoint 3 cleared at 0x49480a for main.main() ./main.go:14
```


在上述示例中, 我们创建并列出三个在 11, 12, 14行的断点, 然后我们马上使用 `clearall` 清除了所有的断点.
在想要快速清除所有断点并调试其他区域的程序时, 这是非常方便的.



## Running and Navigating the program with Delve


当我们做到列出任何部分的源码与设置所需要的断点时, 我们接下来可以看看如何通过一系列强大的命令让代码以调试模式运行并进行真正的调试.

### continue

运行程序直到下一个断点或者程序终止

```
(dlv) break main.go:11
Breakpoint 4 set at 0x4947f8 for main.main() ./main.go:11
(dlv) continue
> main.main() ./main.go:11 (hits goroutine(1):1 total:1) (PC: 0x4947f8)
     6:         return a + b
     7: }
     8:
     9: func main() {
    10:
=>  11:         first := 1
    12:         second := 9
    13:
    14:         result := add(first, second)
    15:
    16:         fmt.Println(result)
```


在 main.go 第 11 行设置一个断点之后, 我们运行 `continue` 命令然后调试器会运行程序直到下一个断点,
在这个例子中即我们刚刚在第 11 行设置的断点.
这个点上我们可以做不少事情, 比如检查或者改变变量的值.
不过首先我们会看下还有什么其他命令来导航程序.


### next

前往下一行源代码
```
(dlv) next
> main.main() ./main.go:12 (PC: 0x494801)
     7: }
     8:
     9: func main() {
    10:
    11:         first := 1
=>  12:         second := 9
    13:
    14:         result := add(first, second)
    15:
    16:         fmt.Println(result)
    17:
```


如此简单! 

`next` 命令让我们只能够在同一时间按照源码中指定位置继续执行一条的指令, 而不管是否其位置上是否存在断点.
这在一步步分析程序时非常有用.

### step

 `step` 或者我喜欢称呼为 `step in` 的命令是告诉调试器进入一个方法的内部调用. 
 它跟 `next` 相似, 不过它会在遇到方法调用时进入一个更深的层次.

```
(dlv) next
> main.main() ./main.go:14 (PC: 0x49480a)
     9: func main() {
    10:
    11:         first := 1
    12:         second := 9
    13:
=>  14:         result := add(first, second)
    15:
    16:         fmt.Println(result)
    17:
    18: }
(dlv) step
> main.add() ./main.go:5 (PC: 0x4947a0)
     1: package main
     2:
     3: import "fmt"
     4:
=>   5: func add(a, b int) int {
     6:         return a + b
     7: }
     8:
     9: func main() {
    10:
```


使用 `step` 命令我们可以进入方法的定义中而不是只计算它的结果然后继续前进.
这在追踪多函数调用的逻辑时很有用.

在不是函数调用行上使用 `step` 命令时, 它表现得就像 `next` 命令.


### stepout

之所以我喜欢将 `step` 称呼为 `stepin` 的原因是它的对立面 `stepout`.

它让我们回到函数调用时所处的位置.


```
(dlv) stepout
> main.main() ./main.go:14 (PC: 0x494819)
Values returned:
        ~r2: 10

     9: func main() {
    10:
    11:         first := 1
    12:         second := 9
    13:
=>  14:         result := add(first, second)
    15:
    16:         fmt.Println(result)
    17:
    18: }
```



### restart

当调试程序已经终止时,如果还想要重新调试可以使用 `restart` 命令.
这样能不丢失所有断点信息, 也不需要退出重新创建一个新的调试会话.

```
(dlv) clearall
Breakpoint 4 cleared at 0x4947f8 for main.main() ./main.go:11
(dlv) continue
10
Process 8652 has exited with status 0
(dlv) restart
Process restarted with PID 9870
```

上面例子中, 我们清除了所有的断点以让程序继续运行到程序终止.
然后使用 `restart` 重新启动进程.
现在我们可以重新开始调试，而无需从头开始创建一个新的会话.




## How to view program variables with Delve

目前为止我们知道了如何添加与管理断点, 如何简易地导航程序运行到任意的位置.
现在我们只需要做到如何查看与编辑程序的变量与内存, 这是调试过程中基础环节与部分.
当然 `delve` 为此提供了许多非常有用的命令来实现这一点.


### print

`print` 是最简单的命令之一, 可以用来查看变量内容与评估表达式.


```
(dlv) break main.go:12
Breakpoint 1 set at 0x494801 for main.main() ./main.go:12
(dlv) continue
> main.main() ./main.go:12 (hits goroutine(1):1 total:1) (PC: 0x494801)
     7: }
     8:
     9: func main() {
    10:
    11:         first := 1
=>  12:         second := 9
    13:
    14:         result := add(first, second)
    15:
    16:         fmt.Println(result)
    17:
(dlv) print first
1
```

在上述例子中, 我们在代码 12 行处设置一个断点并使程序运行到断点处, 打印出变量 `first` 的值.




### locals

`locals` 命令在查检所有局部变量上变得非常有用

```
(dlv) list
> main.main() ./main.go:12 (hits goroutine(1):1 total:1) (PC: 0x494801)
     7: }
     8:
     9: func main() {
    10:
    11:         first := 1
=>  12:         second := 9
    13:
    14:         result := add(first, second)
    15:
    16:         fmt.Println(result)
    17:
(dlv) locals
first = 1
```

## on the road

文章提到的也许只是 `Delve` 冰山一角.
关于命令的精简表示, 关于断点条件的设置,关于查看变量的类型与设置新内容,关于调用栈的信息还有关于协程的调试,关于 Go 单元测试的调试 ...

因为篇幅长度有限没有一一列举, `Delve` 提供了 `help` 以及精简命令 `h` 来提供更加详细的信息, 当遇到特定命令时,可以使用 `help xxx` 来查看更加具体的说明.


## Conclusions

文章介绍的命令相信能够在日常工作满足基础的需求.

同时在 Go 开发的主要集成工具或者编辑器中都有对应的拓展或者插件来进行可视化调试.

如果您掌握了 `Delve` 命令行调试器的使用，那么再使用遵循相同概念和结构的其他调试器将更加容易.



## 参考

- [1] [Golang Debugging with Delve - [Step by Step]](https://golang.cafe/blog/golang-debugging-with-delve.html)
- [2] [Stop debugging Go with Println and use Delve instead](https://opensource.com/article/20/6/debug-go-delve)