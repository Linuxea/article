 
## Overview

- uintptr 是一个无符号整型,大到足够保存所有的指针地址

- 大小是依赖平台

- 仅仅是地址的整数表示



## Properties


- uintptr 可以被转换为 unsafe.Pointer, 反之亦然.
稍后我们会谈论关于 uintptr 转换成 unsafe.Pointer 的有用之处.

- uintptr 可以进行算术运算.这里强调注意下 go 中的指针与 unsafe.Pointer 是不能进行算术运算.

- 即使 uintptr 持有一个指针地址, 但也仅仅是一个值,没有关联任何的对象.因此
 - uintptr 的值在相应的对象移动时并不会更新.
 - uintptr 对应的对象可以被垃圾回收.GC 不会认为 uintptr 是一个存活的引用,并不会影响到对应对象的垃圾回收.

## Purpose

uintptr 可以被用做以下的目的:

- 与 unsafe.Pointer 一起使用于非安全的内存访问. unsafe.Pointer 不能用于算术计算.为了能够执行这样的运算,需要
 - unsafe.Pointer 转换为 uintptr
 - 将运算执行于 uintptr
 - uintptr 转换回 unsafe.Pointer , 通过运算后的地址进行对象访问



应该要关注关于上述的步骤对于 GC 收集器应该是原子的,否则会导致出现问题.

比如第一步操作后,指针关联的对象很容易发生垃圾回收(引用关系不存在).
如果发生在步骤3之后,指针将会是无效并且会导致程序出现崩溃.

关于这一点可能参阅 unsafe 包的文档.
[https://pkg.go.dev/unsafe#Pointer](https://pkg.go.dev/unsafe#Pointer)
它列出了上述的转换在什么时候是安全的.

关于上述提到的场景可以看下以下的代码.
如下代码中,我们进行的运算是获取结构体 `sample` 中字段 `b` 的地址并进行值打印.

以下代码操作是对于垃圾回收器是`原子`的.

```golang
p := unsafe.Pointer(uintptr(unsafe.Pointer(s))) + unsafe.Offsetof(s.b))
```

```golang
package gotour

import (
    "fmt"
    "testing"
    "unsafe"
)

type sample struct {
    a int
    b string
}

func TestUintptr(t *testing.T) {
    s := &sample{a: 1, b: "test"}

    //Getting the address of field b in struct s
    p := unsafe.Pointer(uintptr(unsafe.Pointer(s)) + unsafe.Offsetof(s.b))

    //Typecasting it to a string pointer and printing the value of it
    fmt.Println(*(*string)(p))
}
```

Output
```
test
```




- uintptr 另外一个目的是当为了打印或者存储而保存指针地址，因为地址仅仅只是保存而没有关联任何对象，相应的对象可能被垃圾回收


请参阅下面的代码，我们将 unsafe.Pointer 转换为 uintptr 并打印

另外，请注意前面提到的 unsafe.Pointer 被转换为 uinptr，引用丢失并且引用变量可以被垃圾收集

```golang
func TestUintptr2(t *testing.T) {
    s := &sample{
        a: 1,
        b: "test",
    }
    //Get the address as a uintptr
    pointer := unsafe.Pointer(s)
    startAddress := uintptr(pointer)
    fmt.Printf("Start Address of s: %d\n", startAddress)

}
```

Output

在不同机器上去处的地址很可能不同
```
Start Address of s: 824633984840
```





## 参考

- [1] [Understanding uintptr in Golang](https://golangbyexample.com/understanding-uintptr-golang/)
- [2] [unsafe#Pointer](https://pkg.go.dev/unsafe#Pointer)

























