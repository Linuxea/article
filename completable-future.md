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

设置超时返回默认值的形式来终止任务
```java
completableFuture.completeOnTimeout("default", 3, TimeUnit.SECONDS);
```

join 会阻塞直到任务完成
```java
completableFuture.join();
```

通过这种方式就可以实现手动完成任务


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





