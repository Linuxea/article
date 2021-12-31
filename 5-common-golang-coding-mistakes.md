# 5 Common Golang Coding Mistakes To Avoid


你所应该避免的五个在 Go 编程语言的错误

>一门优秀编程语言的“丑陋”一面

Go 是一门简单且容易学习的编程语言, 同时也是非常快的编程语言, 因为其向下编译为机器码并且拥有一套静态类型系统.

而且 Go 还拥有内置的垃圾回收并且仍然支持着指针,传值与传引用的概念.

这让 Go 非常强大,因为你不会受到大多数编程语言(像 Java 和其他高级编程语言)的限制.

尽管如此, 它同样带来很多导致许多开发者容易犯错的困惑.

在这篇文章中我们会谈谈这些困惑.



## 1. Operation on Slices May or May Not Create a New Underlying Array

对于 slice 切片的操作可能会也可能不会创建新的底层数组

```go
s1 := make([]int, 3, 4)
s2 := s1[2:4]
s3 := append(s1, 0)
s4 := append(s1, 0, 0)
s1[2] = 1
fmt.Println(s1, s2, s3, s4)
--------------
[0 0 1] [1 0] [0 0 1 0] [0 0 0 0 0]
```


切片在 Go 中属于引用类型。它们关联着底层的数组。

当你从一个切片中创建出一个新的切片，实际上两个切片都引用着同一个底层数组。

在上面例子中， `s1`, `s2` 与 `s3` 都是关联着同一底层数组， 所以当 `s1[2]` 更新时，三个切片都会被更新。

然而， 向一个切片添加新元素，原始数组不足以存储更多新数据时可能会导致底层新数组的分配。

这就是 `s4` 的场景。

因此， 我们需要在面对切片时非常小心， 因为底层的数据可能不会按照你所期望的方式改变。


如下的代码可以确保我们在新数组分配时获得其最新的引用。

```golang
type Stack []interface{}
func (stack *Stack) Push(x interface{}) { 
    *stack = append(*stack, x)
}
```






## 2. The Data in the range Clause Are Copies of the Actual Collection Elements

`range` 迭代中实际上是数据复本

```golang
type Point struct {
    X int
    Y int
}
func main() {
    s := make([]Point, 4)
    for i, v := range s {
        v.X = i
        v.Y = i
    }
    fmt.Println(s)
}
--------------
[{0 0} {0 0} {0 0} {0 0}]
```


跟绝大多数内置垃圾回收的编程语言不同的是，Go 中迭代过程中的值出人意料的并不是引用原始的数据条目，而是实际集合元素的一个复本。

因此在上述代码中，更新 `v` 并不会对原始数据造成改变。

为了能够更新原始的值，可以通过索引来操作：
```golang
for i, _ : range s {
    s[i].X = i
    s[i].Y = i
}
```




## 3. Addressable Values vs. Unaddressable Values


可寻址值 vs 不可寻址值

可寻址值在 Go 中是一个棘手的概念。本文中不会过多谈论细节（或许你应该看看这篇[文章](https://utcc.utoronto.ca/~cks/space/blog/programming/GoAddressableValues)），但我们会谈论它是如何改变到我们代码的行为。

```golang
package main

type Point struct {
	X int
	Y int
}

func main() {
	m := make(map[string]Point)
	m["p1"] = Point{1, 1}
	m["p1"].X = 2
```

上面的代码是不能正确工作的， 因为 map 中的值并不是可寻址的并且不能被指定。
相似的错误发生在从一个函数中返回值。


```golang
package main

type Point struct {
	X int
	Y int
}

func main() {

	m := DoIt()
	m["p1"].X = 1

}

func DoIt() map[string]Point {
	m := make(map[string]Point)
	m["p1"] = Point{1, 1}
	return m
}

```


在这种场景下， 你可以使用一个中间变量来做为解决方法.

```golang
m := make(map[string]Point)
m["p1"] = Point{1,1}
p := m["p1"]  // p as an temp variable
p.X = 2
m["p1"] = p // replace with p 
fmt.Println(m)
--------------
map[p1:{2 1}]
```


或者你也可以使用指针 map， 因为指针是间接可寻址的.

```golang
m := make(map[string]*Point)
m["p1"] = &Point{1,1}
m["p1"].X = 2
fmt.Println(m["p1"])
--------------
&{2 1}
```





## 4. Return Pointer to Local Struct

返回指向本地结构的指针 

```golang
func test() *int {
   var i int = 1
   return &i;
}
```

在 C/C++ 编程语言中，上面的代码是不能正常工作的， 因为本地变量是分配在栈上，当方法返回时会被回收。

然而在 Go 语言中，编译器会决定了变量分配的地方。

编译器根据变量的大小与逃逸分析的结果来选择存储变量的位置。

在上面的代码场景中，编译器发现本地变量 `i` 的地址被返回，所以它将变量存储在堆上而不是栈上。

关于更多的详情，你可以阅读这篇[文章](https://www.ardanlabs.com/blog/2017/05/language-mechanics-on-escape-analysis.html)。


理解逃逸分析可以使避免在代码中出现性能问题，变量在栈上与堆的分配在性能表现上是存在差异的。


当 `go build` 或者 `go run` 时使用 gc 标志参数来了解变量分配在哪里（比如 go run -gcflags -m main.go）

```golang
package main

//go:noinline
func main() {

	_ = test()

}

//go:noinline
func test() *int {
	var i = 1
	return &i
}
--------------
go run -gcflags -m main.go
# command-line-arguments
./main.go:12:6: moved to heap: i
```


## 1. nil Checking for Pointers Can Be Confusing

指针的 `nil` 检查可能会造成困惑

```golang
package main

import "fmt"

func main() {

	var i *int
	var np interface{}

	fmt.Println(i, i == nil)
	fmt.Println(np, np == nil)
	np = i
	fmt.Println(np, np == nil)
	// --------------
	// <nil> true
	// <nil> true
	// <nil> false
}
```

在底层，Go 中的接口可以被认为是一个值加上一个具体类型的元组（一个接口保存一个特定底层具体类型的值）。 持有 nil 具体值的接口变量本身是非 nil 的。 如果要检查底层类型的值是否为 nil，可以这样检查：
```golang
np == (*int)(nil) // return true
```



## 参考

- [1] [5 Common Golang Coding Mistakes To Avoid](https://betterprogramming.pub/top-5-common-golang-coding-mistakes-the-ugly-sides-of-a-great-programming-language-e0b64915707)