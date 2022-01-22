## sort workout in golang

>你可能发现在覆盖日常工作中，列表排序是最常见的任务之一。本文即探索在 Go 中使排序简单化的一些技术点。

由于排序这种极其常见的场景，几乎所有的编程语言都会而提供完善的工具来满足需求。

Go 在约定上并无二致，排序包 `sort` 提供所有可能的排序操作。

Go 拥有标准的 `sort` 包来完成排序操作。

排序函数是对数据自身进行排序。

对于基础数据类型，拥有内置的函数。比如：`sort.Ints`, `sort.Strings` 

对于复杂的数据类型，我们需要通过实现 `sort.Interface` 接口来创建我们自己的排序规则。

Go 的 `sort` 包提供内置与用户自定义类型的排序。


## Sort a collection of integers

`sort.Ints` 函数以升序顺序对数据列表进行排序。

```golang
package main

import (
	"fmt"
	"sort"
)

func main() {

	ints := []int{1, 3, 5, 76, 1, 23, 4, 1343, -1, 2}
	fmt.Println(sort.IntsAreSorted(ints)) // false
	sort.Ints(ints) 
	fmt.Println(ints) // [-1 1 1 2 3 4 5 23 76 1343]
	fmt.Println(sort.IntsAreSorted(ints)) // true
}
```

`sort.IntsAreSorted` 报告列表是否以升序的形式有序。


## Sort a collection of strings

`sort.Strings` 函数以升序顺序对数据列表进行排序。

```golang
package main

import (
	"fmt"
	"sort"
)

func main() {

	strings := []string{"ab", "1", "a1", "1a", "bzd", "3434"}
	fmt.Println(sort.StringsAreSorted(strings))
	sort.Strings(strings)
	fmt.Println(strings)
	fmt.Println(sort.StringsAreSorted(strings))
}

```

`sort.StringsAreSorted` 报告列表是否以升序的形式有序。


## Sort a collection of floats

`sort.Float64s` 函数以升序顺序对数据列表进行排序。

```golang
package main

import (
	"fmt"
	"sort"
)

func main() {

	floats := []float64{1.0, 2.0, -1, 64, 32}
	fmt.Println(sort.Float64sAreSorted(floats))
	sort.Float64s(floats)
	fmt.Println(floats)
	fmt.Println(sort.Float64sAreSorted(floats))

}

```


## Sort a collection of structs using an anonymous function

在 Go 中对一个结构体列表进行排序，需要 `less` 函数结合 `sort.Slice` 或者 `sort.SliceStable` 使用。
你需要提供 `less` 函数的实现来对比结构体中的字段。

这里我们演示根据 name 来对 person 列表进行排序：
```golang
package main

import (
	"fmt"
	"sort"
)

func main() {

	type person struct {
		name string
		age  int
	}

	ps := []person{
		{name: "linuxea", age: 18},
		{name: "jay chou", age: 38},
		{name: "jolin", age: 31},
		{name: "lee hong", age: 40},
		{name: "jj", age: 35},
	}

	sort.Slice(ps, func(i, j int) bool {
		return ps[i].age < ps[j].age
	})

	fmt.Printf("%v", ps)

}

```
    
`Less` 函数返回的是索引 i 上的元素是否必须排在索引 j 上的元素。

从 `Less` 函数返回 `true` 会造成索引 i 上的元素排在索引 j 上的元素前面位置。

`sort.Slice` 与 `sort.SliceStable` 之间的区别是后者会保持相等元素的原始顺序，而前者可能不会。



## Sort custom data structures with Len, Less, and Swap

为了就会更加复杂的排序场景，我们需要通过实现 `sort.Interface` 接口来创建自己的排序。


```golang
type Interface interface {
	// Len is the number of elements in the collection.
	Len() int

	// Less reports whether the element with index i
	// must sort before the element with index j.
	//
	Less(i, j int) bool

	// Swap swaps the elements with indexes i and j.
	Swap(i, j int)
}
```

以下的例子来说明实现方式

```golang
package main

import (
	"fmt"
	"sort"
)

type person struct {
	name string
	age  int
}

type persons []person

func (p persons) Len() int {
	return len(p)
}

func (p persons) Less(i, j int) bool {
	return p[i].age < p[j].age
}

func (p persons) Swap(i, j int) {
	p[i], p[j] = p[j], p[i]
}

func main() {

	ps := []person{
		{name: "linuxea", age: 18},
		{name: "jay chou", age: 38},
		{name: "jolin", age: 31},
		{name: "lee hong", age: 40},
		{name: "jj", age: 35},
	}

	sort.Sort(persons(ps))

	fmt.Printf("%v", ps) // [{linuxea 18} {jolin 31} {jj 35} {jay chou 38} {lee hong 40}]

}

```

不过这个例子看起来并不是那么复杂，通过 `func Slice(x interface{}, less func(i, j int) bool)` 也能够实现。


`func Slice(x interface{}, less func(i, j int) bool)` 实现引用了默认的 `Len`  与 `Swap` 实现。


不过可以根据场景实现 `Less`, `Swap` 来将其”复杂“化，这里就不演示了。




## Sort maps keys or values

在 Go 中没有为 `map` 无序结构提供排序。

不过如果确实需要对 `map` 进行排序，而且通过自定义规则将 `keys` 与 `values` 分开来实现。

```golang
package main

import (
	"fmt"
	"sort"
	"strconv"
)

func main() {

	m := map[string]int{
		"1":    1,
		"11":   2,
		"111":  3,
		"1111": 4,
		"2":    5,
		"22":   6,
	}

	fmt.Println(m) // map[1:1 11:2 111:3 1111:4 2:5 22:6]

	keys := make([]string, 0, len(m))
	for key := range m {
		keys = append(keys, key)
	}

	sort.Slice(keys, func(i, j int) bool {
		ii, _ := strconv.Atoi(keys[i])
		jj, _ := strconv.Atoi(keys[j])
		return ii < jj
	})
	for _, temp := range keys {
		fmt.Print(temp, "->", m[temp], " ")
	}
	// 1->1 2->5 11->2 22->6 111->3 1111->4

}
```



## Conclusion

在 Go 中对列表的排序通过 `sort` 包是如此简单明了。

上面提到的场景已经能够覆盖绝大多数开发过程中面临的场景。

## 参考

- [1] [Sorting Workout in Golang](https://medium.com/gitconnected/sorting-workout-in-golang-5b49cbe9154f)
- [2] [go standard lib sort](https://pkg.go.dev/sort)