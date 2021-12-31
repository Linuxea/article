

# template in Go

模板本质上用来生成输出动态内容的静态文本。

就比如一个普通的函数, 使用参数 `name` 来生成不同的字符串 

```javascript
const sayHi = (name) => `Hi my name is ${name}`
sayHi("linuxea")

---------- output
'Hi my name is linuxea'
```


We can also use the same principle to generate web pages in Go. Web templates allow us to serve personalized results to users. Generating web templates using string concatenation is a tedious task. It can also lead to injection attacks.



我们可以使用相同的原则在 Go 生成 web 页面。
模板允许我们向用户提供个性化的结果。

不过常规上使用字符串拼接的方式是枯燥的任务，同时也会导致注入攻击。


# Templates in go

There two packages in go to work with templates:
text/templates (Used for generating textual output)
html/templates (Used for generating HTML output safe against code injection)
Both of them basically have the same interface with subtle web-specific differences in the latter like encoding script tags to prevent them from running, parsing maps as JSON in view, and more.


Go 中有两个与模板相关的包

- text/templates 用来生产文本输出
- html/templates 用来生产避免代码注入风险的 html 文本

两者之间具有相同的接口与微妙的区别。比如对 script 标签进行编码来防止其运行， 解析 map 为 json 数据等等。



# Our first Template

```golang
package main

import (
	"html/template"
	"os"
)

func main() {

	t := template.Must(template.New("firstTp").Parse(`Hi my name is {{.}}`))
	err := t.Execute(os.Stdout, "linuxea")
	if err != nil {
		panic(err)
	}

}
```
The extension of a template file could be anything. We are using .gohtml so that it is supported development tools across IDEs.

“Actions” — data evaluations or control structures — is delimited by{{and}}. The data evaluated inside it is called a Pipeline. Anything outside them is sent to the output unchanged.

- 