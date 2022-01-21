## sort workout in golang


>Sorting the list of data is one of the most common tasks you may find yourself coding ever so often. This article explores a few techniques in Golang that should make the sorting of any data collection simple.

>你可能发现在覆盖日常工作中，数据列表排序是最常见的任务之一。本方探索在 Go 中使排序简单化的一些技术点。


All programming languages often provide ready utils to sort collections due to the incredibly common nature of this task in solving coding assignments. Go is no different with its provision of the sort package for performing all possible sorting operations.


由于排序这种极其常见的场景，几乎所有的编程语言都会而提供完善的工具来满足需求。


Go 在条款约定上并无二致，排序包 `sort`提供所有可能的排序操作。

Go has the standard sort package for doing the sorting. The sorting functions sort data in place. For basic data types, we have built-in functions such as sort.Ints and sort.Strings. For more complex types, we need to create our own sorting by implementing the sort.Interface. Golang’s sort the package implements sorting for builtins and user-defined types.


Go 拥有标准的 `sort` 包来完成排序操作。
排序函数对数据自身进行排序。

对于基础数据类型，拥有内置的函数（比如：sort.Ints, sort.Strings） 
对于复杂的数据类型，我们可能需要通过实现 `sort.Interface` 接口来创建我们自己的排序规则。

Go 的 `sort` 包实现了内置与用户定义类型的排序。


## Sort a collection of integers

The sort.Ints function sorts a slice of integers in ascending order.

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

The sort.IntsAreSorted the method also exists to check if an integer slice is in its sorted form or not.

`sort.IntsAreSorted` 报告列表是否以升序的形式有序


## Sort a collection of strings

The sort.Strings function sorts a slice of strings in ascending order.

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

The sort.StringsAreSorted the method also exists to check if a string slice is in its sorted form or not.

`sort.StringsAreSorted` 报告列表是否以升序的形式有序


## Sort a collection of floats

The sort.Float64s function sorts a slice of floating-point values in ascending order.

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


To sort a slice of structs in Golang, you need to use a less function along with either the sort.Slice or sort.SliceStable methods. You need to provide the implementation of less functions to compare structure fields. Here’s how we can sort a slice of persons according to their names and ages for example:


在 Go 中对一个结构体列表进行排序，需要 `less` 函数结合 `sort.Slice` 或者 `sort.SliceStable` 使用。
你需要提供 `less` 函数的实现来对比结构体中的字段。

这里我们演示根据 name 与 age 来对一些 person 列表进行排序：
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

	// Less reports whether the element with index i
	// must sort before the element with index j.
    
`Less` 函数返回的是索引 i 上的元素是否必须排在索引 j 上的元素。

Returning true from the less function will cause the element at the index i to be sorted to a lower position than the index j (The element at index i will come first in the sorted slice). Otherwise, the element at the index j will come first if false is returned.

从 `Less` 函数返回 `true` 会造成索引 i 上的元素排在索引 j 上的元素前面位置。


The difference between sort.Slice and sort.SliceStable is that the latter will keep the original order of equal elements while the former may not.

`sort.Slice` 与 `sort.SliceStable` 之间的区别是后者会保持相等元素的原始顺序，而前者可能不会。



## Sort custom data structures with Len, Less, and Swap

For more complex types, we need to create our own sorting by implementing the sort.Interface:

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

The below example illustrates this type of implementation.

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

Maps are unordered collections in Golang so there is no provision for sorting a map using the sort package. However, if you really need to sort a map, you can write your custom implementation for it by slicing out keys & values separately.


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

Sorting collections of data are pretty straightforward in Golang through the use of the sort package and the solutions presented above should suffice for the majority of situations you will face.

在 Go 中对列表的排序通过 `sort` 包是如此简单明了。

上面提到的场景已经能够覆盖绝大多数开发过程中面临的场景。

## 参考

- [1] [Sorting Workout in Golang](https://medium.com/gitconnected/sorting-workout-in-golang-5b49cbe9154f)
- [2] [go standard lib sort](https://pkg.go.dev/sort)