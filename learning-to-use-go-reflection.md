# What's Reflection ?


绝大多数情况下,变量,类型与方法在 go 中的使用都是直接了当.

当你需要一个类型时,你可以定义一个类型:
```golang
type Foo struct {
 A int
 B string
}
```

当你需要一个变量时,你可以定义一个变量:
```golang
var x Foo
```


当你需要一个方法时,你可以定义一个方法:
```golang
func DoSomething(f Foo) {
	fmt.Println(f.A, f.B)
}
```


但是有时候,你可能会处理程序完成时还没有存在的变量.
- 可能你想要从一个文件或者网络请求中将数据映射到一个变量.
- 可能你想要构建一个可以处理不同类型变量的工具.

在这些场景下,你需要用到反射.
反射给予你运行时检查类型的能力,允许你能够检查,创建变量,函数,结构体.


反射主要包含三个概念: `Types`, `Kinds`, 和 `Values`.
`reflect` 包是 go 中实现反射操作的标准包.

## Finding Your Type

首先我们来看看第一个概念 types.
你可以使用反射api `varType := reflect.TypeOf(var)` 来获取一个变量的类型,它返回一个类型为 `reflect.Type` 的变量,它包含了传递给它的参数的所有信息与定义.

第一个方法是 `Name()`.毫无意外返回的是类型的名称.有些类型,比如切片,指针,因为没有名称,所以返回的是一个空字符串.


接下来的方法,也是我所认为第一个有用的方法是 `Kind()`.
kind 解释了 type 是由什么组成的- slice, map, pointer, struct, interface, string, array, function, int 或者其他原始类型.

kind 与 type 的区别可能难以理解,不过可以这样想:
你定义了一个结构体叫做 `Foo`, kind 表示为 `struct`, type 表示为 `Foo`.


当使用反射时需要意识到,反射包的 api 都是假设你知道你正在做什么,许多的函数或者方法如果使用不当会触发 panic.
比如调用的 reflect.Type 方法与其类型不一致.


如果你的变量是一个指针,map, slice, channel 或者数组,你可以使用 varType.Elem() 来找出容器类型.

如果你的变量是一个结构体,你可以使用反射来获取结构体中字段的数量,并获取包含在 `reflect.StructField` 结构体中每个字段的结构.
reflect.StructField 包含字段的名称,顺序,类型与 tags 标记.


>Talk is cheap, show me the code

```golang
type Foo struct {
	A int `tag1:"First Tag" tag2:"Second Tag"`
	B string
}

func main() {
	sl := []int{1, 2, 3}
	greeting := "hello"
	greetingPtr := &greeting
	f := Foo{A: 10, B: "Salutations"}
	fp := &f

	slType := reflect.TypeOf(sl)
	gType := reflect.TypeOf(greeting)
	grpType := reflect.TypeOf(greetingPtr)
	fType := reflect.TypeOf(f)
	fpType := reflect.TypeOf(fp)

	examiner(slType, 0)
	examiner(gType, 0)
	examiner(grpType, 0)
	examiner(fType, 0)
	examiner(fpType, 0)
}

func examiner(t reflect.Type, depth int) {
	fmt.Println(strings.Repeat("\t", depth), "Type is", t.Name(), "and kind is", t.Kind())
	switch t.Kind() {
	case reflect.Array, reflect.Chan, reflect.Map, reflect.Ptr, reflect.Slice:
		fmt.Println(strings.Repeat("\t", depth+1), "Contained type:")
		examiner(t.Elem(), depth+1)
	case reflect.Struct:
		for i := 0; i < t.NumField(); i++ {
			f := t.Field(i)
			fmt.Println(strings.Repeat("\t", depth+1), "Field", i+1, "name is", f.Name, "type is", f.Type.Name(), "and kind is", f.Type.Kind())
			if f.Tag != "" {
				fmt.Println(strings.Repeat("\t", depth+2), "Tag is", f.Tag)
				fmt.Println(strings.Repeat("\t", depth+2), "tag1 is", f.Tag.Get("tag1"), "tag2 is", f.Tag.Get("tag2"))
			}
		}
	}
}
```

## Making a New Instance

除了检查变量的类型,使用反射还可以读,写和创建变量.
首先你需要 reflect.ValueOf(var) 为变量创建 reflect.Value 实例, 如果你想要能够对变量进行修改,那必须获取关于变量指针的 reflect.Value 实例:`refPtrVal = reflect.ValueOf(&var)`,
否则你只能对读取而不能修改.

一旦你拥有了 reflect.Value 实例,你可以通过 `Type()` 方法来获取 `reflect.Type` .

如果你想要修改一个值,记住必须使用指针,并且必须先对它进行解引用.
`refPtrVal.Elem().Set(newRefVal)`, 传递给 Set 的参数也必须是 reflect.Value 类型


如果你想要创建一个值,可以使用 `reflect.New(varType)`, 传递参数为 reflect.Type.返回的是一个指针值.




最后,如果你想要回退到一个普通的变量,可以通过调用 `Interface` 方法.
因为 Go 没有泛型,所以原始的类型信息已经丢失,因此方法返回的是一个 interface{} 空类型.
如果你创建的是一个指针以便进行修改,首先需要进行解引用 `Elem().Interface()`.

不过哪种情况,你都需要在程序中将返回的空接口转换为你想要真正使用的类型.
```golang
package main

import (
	"fmt"
	"reflect"
)

type person struct {
	Name string
	Age  int
}

func main() {

	p := person{Name: "linuxea", Age: 19}

	typeOf := reflect.TypeOf(p)
	newValue := reflect.New(typeOf)

	newValue.Elem().Field(0).SetString("linuxea-copy")
	newValue.Elem().Field(1).SetInt(20)

	if p2, ok := newValue.Elem().Interface().(person); ok {
		fmt.Println("HHHHHH, ", p2)
	}

}

```




## Making Without Make

创建内建和用户自定义类型时,可以使用反射而非通常要求的 make 方法.
- 创建一个 slice, `reflect.MakeSlice`
- 创建一个 map, `reflect.Makemap`
- 创建一个 channel, `reflect.MakeChan`

在所有场景下,你提供一个 reflect.Type 并且获取一个 reflect.Value, 或者可以指定回退到一个标准变量.





## Making Functions
反射并不仅仅用来让你创建存放数据.
你也可以用`reflect.MakeFunc` 函数来创建新的函数.

它接收类型为 reflect.Type 的函数,输入参数与输出参数都是 []reflect.Value.
```golang
package main

import (
	"fmt"
	"reflect"
	"time"
)

func doHomework() {
	fmt.Println("我在做家务啊,做啊做")
	time.Sleep(time.Second * 3) // 做个3秒
	fmt.Println("我家务做完了,去玩了")
}

func main() {

	// make functions
	doHomeworkType := reflect.TypeOf(doHomework)
	doHomeworkValue := reflect.ValueOf(doHomework)
	makeFunc := reflect.MakeFunc(doHomeworkType, func(value []reflect.Value) []reflect.Value {
		now := time.Now()
		call := doHomeworkValue.Call(value)
		fmt.Println(fmt.Sprintf("总共花费时:%d", time.Now().Unix()-now.Unix()))
		return call
	})
	if f, ok := makeFunc.Interface().(func()); ok {
		f()
	}

}

```


## I Want a New struct

使用反射还可以做一件事,通过传递元素类型为 reflect.StructField 的切片给 reflect.StructOf 函数,
可以构建一个全新的结构体.
有一点奇怪的是,我们不能给这个结构体命名,所以我们没法将它回退成一个正常的变量.
你可能创建一个新实例并使用 interface() 来把它保存到类型为空接口变量中,但是如果你想要给结构体设置值,需要用到反射.


```golang
package main

import (
	"fmt"
	"reflect"
)


func MakeStruct(vals ...interface{}) interface{} {
	var sfs []reflect.StructField
	for k, v := range vals {
		t := reflect.TypeOf(v)
		sf := reflect.StructField{
			Name: fmt.Sprintf("F%d", k + 1),
			Type: t,
		}
		sfs = append(sfs, sf)
	}
	st := reflect.StructOf(sfs)
	so := reflect.New(st)
	return so.Interface()
}

func main() {
	s := MakeStruct(0, "", []int{})
	// this returned a pointer to a struct with 3 fields:
	// an int, a string, and a slice of ints
	// but you can’t actually use any of these fields
	// directly in the code; you have to reflect them
	sr := reflect.ValueOf(s)

	// getting and setting the int field
	fmt.Println(sr.Elem().Field(0).Interface())
	sr.Elem().Field(0).SetInt(20)
	fmt.Println(sr.Elem().Field(0).Interface())

	// getting and setting the string field
	fmt.Println(sr.Elem().Field(1).Interface())
	sr.Elem().Field(1).SetString("reflect me")
	fmt.Println(sr.Elem().Field(1).Interface())

	// getting and setting the []int field
	fmt.Println(sr.Elem().Field(2).Interface())
	v := []int{1, 2, 3}
	rv := reflect.ValueOf(v)
	sr.Elem().Field(2).Set(rv)
	fmt.Println(sr.Elem().Field(2).Interface())
}

```





## What Can't You Do ?

反射有一个大的局限.
尽管通过反射可以创建新的函数,却没有办法在运行时创建一个方法.
这意味着你不能利用反射在运行时实现接口.

```golang
package main

import "fmt"

type foo struct {
	age int
}

type bar struct {
	foo
}

type Doubler interface {
	Double() int
}

func (f foo) Double() int {
	return f.age * 2
}

func DoDouble(d Doubler) {
	fmt.Println(d.Double())
}

func main() {

	f := foo{
		age: 10,
	}
	DoDouble(f)

	b := bar{
		foo: f,
	}
	DoDouble(b)

}

```

- Bar 的 Foo 字段没有名称,是一个匿名或者说嵌入字段
- Bar 被认为实现了 Doubler 接口,尽管实现了 Doubler 接口的只有 Foo

这种能力被叫做委托.
在编译时期,Go 会自动在 Bar 生成办法来匹配其匿名或者嵌入的 Foo.
这不是继承,如果你把 Bar 当做参数传递给预期参数为 Foo 的方法,你的代码是不会编译通过的.




## 参考

- [1] [Learning to Use Go Reflection](https://medium.com/capital-one-tech/learning-to-use-go-reflection-822a0aed74b7)


















