# embed

`embed` 包提供 Go 代码运行时访问嵌入文件的能力。


引入了 `embed` 包的 Go 代码文件，可以使用 `//go:embed` 指令，在编译期读取包目录或者子目录下的文件内容来初始化类型为 `string`, `[]byte`, `FS` 类型的变量。

举个粟子，列出三种方式来嵌入 `hello.txt`  文件并且在运行时打印其内容。


嵌入文件内容到 string ：
```golang
import _ "embed"

//go:embed hello.txt
var s string
print(s)
```

嵌入文件内容到 byte[]：
```golang
import _ "embed"

//go:embed hello.txt
var b []byte
print(string(b))
```

嵌入一个或多个文件内容到文件系统变量 FS ：
```golang
import _ "embed"

//go:embed hello.txt
var f embed.FS

data, _ ;= f.ReadFile("hello.txt")
print(string(data))
```


## Directives

声明变量上的 `//go:embed` 指令指定了哪些文件进行嵌入， 使用了一个或多个的 `path.Match` 路径匹配模式。


`//go:embed` 指令必须紧跟着包含单个变量声明的一行。只有`空格`和 `//注释` 允许出现在指令与变量声明中间。

变量的类型必须为 `string`, `byte[]` 或者 `FS`。


比如：
```golang
package server

import "embed"

// content holds out static web server content.
//go:embed image/* template/*
//go:embed html/index.html
var content embed.FS
```

Go 构建系统将会识别出指令信息并为变量安排填充为文件系统匹配的文件。

为了配置简洁，`//go:embed` 指令接受以空格分隔的多模式，但是指令也是可重复的，这是为了避免在多模式下造成当前行过长问题。

这些模式是相对于包含源文件的包目录进行解释的。

路径分隔符是一个正斜杠，即使在 Windows 系统上也是如此。

模式不能包含 `.` 或者 `..` 或者空路径元素，也不能以 `/` 开始或结束。

为了匹配当前目录的所有文件，使用 `*` 而不是 `.`. 

为了允许命名带有空格的文件，模式可以写为`双引号`或`反引号`字符串文字。

如果模式命名为一个目录名称，则以该目录为根的子树中所有文件都会被嵌入（递归），除了以 `.` 或者 `_` 开头的文件会被排除。

因此，上面例子中的变量大致等同于：
```golang
// content is our static web server content.
//go:embed image template html/index.html
var content embed.FS
```

区别在于 `image/*` 会嵌入 `image/.tempfile` 文件而 `image` 则不会，`image/dir/.tempfile` 文件也是如此。


在以 `all:` 为前缀开始的模式中，目录的遍历规则变更为包含以 `.` 或者 `_` 开始的文件。比如，`all:image` 包含了 `image/.tempfile` 与 `image/dir/.tempfile`。

`//go:embed` 指令可以导出或非导出变量一起使用，这取决于包是否想让数据对其他包可见。

指令只可以用在包级变量，不可以用于方法本地变量。


模式不能匹配到包模块外的文件，比如 `.git/*` 或者符号链接。匹配空目录会被忽略。除此，`//go:embed` 行中的每个模式必须匹配至少一个文件或非空目录。

如果存在无效的模式，构建将会失败。


## Strings and Bytes

类型为 string 或者 []byte 的变量的 //go:embed 指令行只能有一个模式，并且该模式只能匹配到一个文件。

变量会初始化为文件的内容。

//go:embed 指令要求引入 `embed` 包，即使我们使用的是 string 或者 []byte。在不引用 embed.FS 的源文件中，使用空白导入 (`import _ "embed"`)。



## File Sytems

对于嵌入一个文件来说，string 或者 []byte 的变量往往是最好的选择。

FS 类型能够嵌入文件树，比如上个例子中静态 web 服务内容。

FS 实现了 `io/fs` 包的 `FS` 接口，因此它可以与理解文件系统的任何包一起使用，包含 `net/http`, `text/template` 与 `html/template`.

举个例子，给定上述例子变量的内容，我们可以这样写：
```golang
http.Handle("/static/", http.StripPrefix("/static/", http.FileServer(http.FS(content))))

template.ParseFS(content, "*.tmpl")
```


## Tools

为了支持分析 Go 的工具，//go:embed 中找到的模式在 `go list` 输出中是可见的。通过 `go help list` 来查看 `EmbedPatterns`， `TestEmbedPatterns`， `XTestEmbedPatterns` 的字段信息。（比如 ` go list -f '{{.EmbedPatterns}}'`）



## 参考

- [1] [embed](https://pkg.go.dev/embed)
- [2] [Go: Introduction to Embed Package](https://towardsdev.com/go-introduction-embed-package-a086c8e53753)