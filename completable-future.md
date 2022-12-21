# Introduction In CompletableFuture


> Java 8 came up with tons of new features and enhancements like Lambda expressions, Streams, CompletableFutures etc. In this post I’ll give you a detailed explanation of CompletableFuture and all its methods using simple examples.


> Java 1.8 带来了如 Lambda 表达式，Stream 等丰富的功能，除此之外，还有 CompletableFuture


## What’s a CompletableFuture?

CompletableFuture 是 Java 的异步编程框架。

 Asynchronous programming is a means of writing non-blocking code by running a task on a separate thread than the main application thread and notifying the main thread about its progress, completion or failure.


异步编程是一种通过在与主应用程序线程不同的线程上运行任务并通知主线程其进度、完成或失败来编写非阻塞代码的方法。
主线程不会阻塞/等待任务完成，它可以并行执行其他任务。


拥有这种并行性可以极大地提高程序的性能。


### Limitations of Future

A Future represents the result of an asynchronous computation. 

jdk1.5 引入的 `Future` 代表异步计算的结果。

`CompletableFuture` 相比 `Future`，在异步编程方式与功能上都更进一步。
- 支持手动完成任务
- 非阻塞式方式继续执行下一个任务
- 多个任务支持链接串联
- 支持多个任务整合捆绑
- 支持异常处理



## Creating a CompletableFuture

使用空构造器创建一个任务为 String 的最简单的 CompletableFuture

```java
CompletableFuture<String> completableFuture = new CompletableFuture<>();
```

> 1. 支持手动完成任务

设置超时返回默认值的形式来终止任务
```java
completableFuture.completeOnTimeout("default", 3, TimeUnit.SECONDS);
```

join 会阻塞直到任务完成
```java
completableFuture.join();
```


## Running asynchronous computation using runAsync()

If you want to run some background task asynchronously and don’t want to return anything from the task, then you can use CompletableFuture.runAsync() method. It takes a Runnable object and returns CompletableFuture<Void>.


`runAsync()` 是以异步方式运行 `Runnable`.

```java
CompletableFuture.runAsync(() -> {
    System.out.println("I am done");
    }
}).join();
```



## Run a task asynchronously and return the result using supplyAsync()


相比 `runAsync()` ，如果想从异步任务中获取一个结果，`supplyAsync()` 是你的选择。
参数是 `Supplier`

```java
CompletableFuture.supplyAsync(() -> 1);
```

> `Supplier<T>` 是最常用的函数式接口之一， 用来提供使用者一个类型为 `T` 的值


## Transforming and acting on a CompletableFuture

直接使用 `get` 或者 `join` 会阻塞当前线程直到任务完成。

但这是我们想要的吗。

对此，`CompletableFuture` 提供了回调机制来构造异步系统。

我们不需要等待任务完成，只需要在回调函数中完成编写任务结果的处理逻辑。

> 2.非阻塞式方式继续执行下一个任务

你可以使用 `thenApply()`、`thenAccept()` 和 `thenRun()` 方法将回调方法附加到 `CompletableFuture`


### thenApply()

使用 `thenApply()` 处理任务结果并进行转化。

It takes a Function<T,R> as an argument. Function<T,R> is a simple functional interface representing a function that accepts an argument of type T and produces a result of type R -

它使用了 `Function<T,R>` 作为参数。

> Function<T,R> 也是最常用的函数式接口之一，用来对类型为 T 的输入参数进行转换为类型为 R 的输出结果

```java
// 将任务结果 linuxea 转化为对其求字符串长度
CompletableFuture.supplyAsync(() -> "linuxea").thenApply(String::length).join()
```

可以通过附加多个 `thenApply()` 方法来编写一系列转换。上一个 `thenApply()` 的结果会传递给下一个 `thenApply()`。

```java
CompletableFuture.supplyAsync(() -> "linuxea") // 任务结果返回 linuxea 字符串
        .thenApply(String::length) // 对任务结果取字符串长度
        .thenApply(length -> length > 5) // 判断长度是否大于 5
        .thenApply(String::valueOf) // 将 Boolean 转化成字符串形式
        .join();
```

## thenAccept() and thenRun()

与 `thenApply()` 相比，如果你不想从回调中返回任务结果，而只是想运行借由上一个任务的结果完成一些代码逻辑，`thenAccept()` 是你的选择。

`thenAccept()` 参数是一个 `Consumer<T>`。

```java
CompletableFuture.supplyAsync(() -> "linuxea") // 任务结果返回 linuxea 字符串
        .thenAccept(System.out::println) // 直接把任务结果进行打印
    ;
```

> `Consumer<T>` 也是最常用的函数式接口之一，对类型为 T 的参数进行消费且不返回任何结果值


If you don’t want to return anything from your callback function and just want to run some piece of code after the completion of the Future, then you can use thenAccept() and thenRun() methods. These methods are consumers and are often used as the last callback in the callback chain.


与 `thenAccept()` 相似的是 `thenRun()` 。

`thenRun()` 的参数是 `Runnable`。它不需要上一个任务的结果，也不返回任何结果。当只需要通知某些人任务已经完成而不需要具体的任务结果时，它会派上用场。

```java
CompletableFuture.supplyAsync(() -> "linuxea") // 任务结果返回 linuxea 字符串
        .thenRun(() -> System.out.println("I am done")) // 通知任务完成
```


> `Runnable` 也是我们最熟悉的函数式接口。它没有参数也没有返回值


## Combining two CompletableFutures together

### thenCompose()

将有依赖关系的任务通过 `thenCompose()` 组合起来

```java
public CompletableFuture<String> getUserName(Long uid) {
    //忽略参数直接返回用户名
    return CompletableFuture.supplyAsync(() -> "linuxea");
  }

  public CompletableFuture<String> getIDCard(String userName) {
    //忽略参数直接返回 id card
    return CompletableFuture.supplyAsync(() -> "1234567890");
  }
```

使用 `thenCompose(CompletableFuture)` 组合两个 `CompletableFuture`
```java
getUserName(uid).thenCompose(this::getIDCard).join()
```

如果使用上面提到的 `thenApply()` 会有什么不同吗。同样是把一个结果映射为另外一个结果。

```java
CompletableFuture<CompletableFuture<String>> completableFutureCompletableFuture = this.getUserName(uid).thenApply(this::getIDCard);
```

我们返回的是一个 `CompletableFuture` 嵌入 `CompletableFuture`。
如果我们想要一个扁平化的 `CompletableFuture`那使用 `thenCompose`。

### thenCombine()

`thenCompose()` 整合了具有依赖关系的 `CompletableFuture`。

`thenCombine()` 是将两个独立的 `CompletableFuture` 并行处理并在各自任务时进行整合。

```java
//取姓名 future
CompletableFuture<String> name = CompletableFuture.supplyAsync(() -> "linuxea");
//取年龄 future
CompletableFuture<Integer> age = CompletableFuture.supplyAsync(() -> 18);

// combine two independent future
name.thenCombine(age, (nameResult, ageResult) -> nameResult + "is age is " + ageResult).join();
//or 
age.thenCombine(name, (nameResult, ageResult) -> nameResult + "is age is " + ageResult).join();
```

当两个 `CompletableFuture` 都完成时，将调用传递给 `thenCombine()` 的回调函数。


## Combining multiple CompletableFutures together

We used thenCompose() and thenCombine() to combine two CompletableFutures together. Now, what if you want to combine an arbitrary number of CompletableFutures? Well, you can use the following methods to combine any number of CompletableFutures -

我们使用 `thenComponse()` 与 `thenCombine()` 来合并两个 `CompletableFutures`。

如果我们想要合并任意数量的 `CompletableFutures` 呢？

好吧。你可以使用如下方法来实现：

