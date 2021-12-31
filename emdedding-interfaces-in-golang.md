

# Embedding Interfaces in Golang

> 一个接口可以被嵌入到另外一个接口中，也可以被嵌入到一个结构体中


## Embedding interface in another interface

一个接口可以嵌入任意数据的接口，也可以被嵌入到另外任意一个接口
所有被嵌入的接口的方法都会成为嵌入接口的一部分。

从而可以用来实现将多个小接口合并成一个大接口

让我们来看看一个例子吧：

假设我们有一个叫做 `animal` 的接口如下：

todo



有另外一个命名为 `human` 的接口，它嵌入了 animal 接口

todo


所以如果有类型想要实现 human 接口， 它必须定义所有的接口方法:
- 作为被嵌入接口的方法 `breathe()` , `walk()` 
- human 接口自己的方法 `speak()`

todo



`ReaderWriter` 可以作为另外的例子，它是 golang io 包中的接口 [ReaderWriter](https://pkg.go.dev/io#ReadWriter)

- [Reader interface](https://pkg.go.dev/io#Reader)
- [Writer interface](https://pkg.go.dev/io#Writer)




## Embedding interface in a struct


接口同样可以被嵌入到结构体当中

可以通过结构体来访问被嵌入接口的所有方法

至于怎么调用这些方法取决于嵌入接口是命名字段还是匿名字段。

- 如果是命名字段，那接口的方法必须通过命名名称来调用
- 如果是非命名字段或者匿名字段，则可以通过结构体直接调用或者通过接口名称


todo



如果我们对结构体中的接口字段进行实例化，默认会被初始化为零值即 nil，
直接使用会触发 panic 异常


### 参考
- [1][Embedding Interfaces in Go (Golang)](https://golangbyexample.com/embedding-interfaces-go/)








