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

```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
```

###  allOf

`allOf` 使用场景是在并行完成一系列独立的 `CompletableFuture` 后，你可以实现后续处理。

```java
//随便找几个网站
List<String> urls = new ArrayList<>(){{
    add( "https://www.zhihu.com/question/571018901/answer/2793033135");
    add("https://www.zhihu.com/question/308641794/answer/2809281113");
    add("https://www.zhihu.com/question/340109086/answer/2774107425");
    add("https://zhuanlan.zhihu.com/p/86351416");
    add("https://www.zhihu.com/question/557506449/answer/2707643182");
    add("https://www.zhihu.com/question/30196513/answer/2806480067");
}};
    

// 根据每个网页生成各自的爬取任务 CompletableFuture
List<CompletableFuture<String>> futures = urls.stream()
    .map(url -> CompletableFuture.supplyAsync(() -> this.fetchNet(url)))
    .collect(Collectors.toList());

// allOf 返回的是 CompletableFuture<Void>
CompletableFuture<Void> allOf = CompletableFuture.allOf(
    futures.toArray(CompletableFuture[]::new));

//调 allOf CompletableFuture 直到所有的任务都完成
allOf.join();

//调用每个子 CompletableFuture 获取任务结果
return futures.stream().map(CompletableFuture::join).collect(Collectors.toList());


// 根据 url 进行网页爬取的方法
public String fetchNet(String url){
    HttpRequest request = HttpRequest.newBuilder()
        .uri(new URI(url))
        .GET()
        .timeout(Duration.of(3, SECONDS))
        .build();
    HttpResponse<String> response = HttpClient.newHttpClient().send(request, BodyHandlers.ofString());
    return response.body();
}
```


### anyOf

CompletableFuture.anyOf() as the name suggests, returns a new CompletableFuture which is completed when any of the given CompletableFutures complete, with the same result.

`CompletableFuture.anyOf()` 如字面上意思，在给定的多个 `CompletableFutures` 中任一完成时返回一个同样结果的新的 `CompletableFuture`。


修改上述部分代码

```java
// anyOf
CompletableFuture<Object> anyOf = CompletableFuture.anyOf(futures.toArray(CompletableFuture[]::new));
// 等待直到返回第一个也是唯一一个任务结果
anyOf.join();
```

`anyOf()` 的问题在于如果给定的多个 `CompletableFutures` 拥有不同的返回类型，那就不知道最终的 `anyOf()` 结果类型是什么。
```java
CompletableFuture<Object> diffTypeCompletableFutures = CompletableFuture.anyOf(
        CompletableFuture.supplyAsync(() -> "abc"), CompletableFuture.supplyAsync(() -> 1));
// what type am i?
Object join = diffTypeCompletableFutures.join();
```


## CompletableFuture Exception Handling

We explored How to create CompletableFuture, transform them, and combine multiple CompletableFutures. Now let’s understand what to do when anything goes wrong.

我们已经探索了如何创建 `CompletableFuture`, 以及转化结果，合并多个 `CompletableFutures` 的方法。

现在我们来了解下当事情出现异常时该做什么。

Let’s first understand how errors are propagated in a callback chain. Consider the following CompletableFuture callback chain -

我们首先明白错误在回调链是如何传播的。先考虑如下代码：
```java
CompletableFuture.supplyAsync(() -> {
	// Code which might throw an exception
	return "Some result";
}).thenApply(result -> {
	return "processed result";
}).thenApply(result -> {
	return "result after further processing";
}).thenAccept(result -> {
	// do something with the final result
});
```

If an error occurs in the original supplyAsync() task, then none of the thenApply() callbacks will be called and future will be resolved with the exception occurred. If an error occurs in first thenApply() callback then 2nd and 3rd callbacks won’t be called and the future will be resolved with the exception occurred, and so on.

创建一个 `Future` 时，如果异常发生在一开始的 `supplyAsync()` 阶段，那没有任何一个 `thenApply()` 回调逻辑被调用，同时 `future` 将面临处理这个异常。

如果异常出现在第一个 `thenApply()`, 那么第二跟第三个 `thenApply()` 将不会触发回调，同样 `future` 将面临处理这个异常。

以此类推。


### 1. Handle exceptions using exceptionally() callback

The exceptionally() callback gives you a chance to recover from errors generated from the original Future. You can log the exception here and return a default value.


```java
/**
    * Returns a new CompletableFuture that is completed when this
    * CompletableFuture completes, with the result of the given
    * function of the exception triggering this CompletableFuture's
    * completion when it completes exceptionally; otherwise, if this
    * CompletableFuture completes normally, then the returned
    * CompletableFuture also completes normally with the same value.
    * Note: More flexible versions of this functionality are
    * available using methods {@code whenComplete} and {@code handle}.
    *
    * @param fn the function to use to compute the value of the
    * returned CompletableFuture if this CompletableFuture completed
    * exceptionally
    * @return the new CompletableFuture
    */
public CompletableFuture<T> exceptionally(
    Function<Throwable, ? extends T> fn) {
    return uniExceptionallyStage(fn);
}
```

`exceptionally` 在上一步给定的函数触发异常时返回新的 `CompletableFuture`。
否则返回跟上一步相同值的 `CompletableFuture`。

定义一个 `IntSupplier` 实例, 实际触发异常。
```java
Supplier<Integer> supplier = () -> {throw new RuntimeException("error is me");};
```

```java
CompletableFuture.supplyAsync(supplier)
    .exceptionally(throwable -> {
        // log
        System.out.println("something wrong happens" + throwable.getMessage());
        // default value
        return -1;
    })
    .thenApply(integer -> integer / 0)
    .exceptionally(throwable -> {
        // log
        System.out.println("something wrong happens again" + throwable.getMessage());
        // default value
        return -2;
    })
    .join();
```

一旦处理了对应的异常就不会在回调链上继续传播。
最终结果如下：
```console
something wrong happensjava.lang.RuntimeException: error is me
something wrong happens againjava.lang.ArithmeticException: / by zero
-2
```

### 2. Handle exceptions using the consumer whenComplete method

对异常的具体处理可以使用 `whenComplete()`, 它的参数是 `BiConsumer<? super T, ? super Throwable>`，
通过上一步的结果与异常实现相应的消费逻辑。它不具备更改结果值的 `handle` 能力。

```java
Supplier<String > supplier = () -> { throw new RuntimeException("error is me");};
CompletableFuture.supplyAsync(supplier)
    .whenComplete((s, throwable) -> {
        System.out.println("has a error ?:" + (throwable != null));
        System.out.println("future result is " + s);
    })
    .exceptionally(throwable -> "default");
```


### 3. Handle exceptions using the generic handle() method

相比之下，`handle()` 是对付异常更加通用的处理方式。

```java
Supplier<String > supplier = () -> { throw new RuntimeException("error is me");};
CompletableFuture.supplyAsync(supplier)
    .handle(((s, throwable) -> {
        if(throwable != null) {
        System.out.println("error occurs" + throwable.getMessage());
        return "error";
        }
        return s;
    })).join();
```



Conclusion
Congratulations folks! In this tutorial, we explored the most useful and important concepts of CompletableFuture API.

Thank you for reading. I hope this blog post was helpful to you. Let me know your views, questions, comments in the comment section below.


## Conclusion

在上述示例中，介绍了 `CompletableFuture` 部分重要且有用的概念。
还有部分异步 `API` 没有介绍到，比如转换时以异步方式处理有 `thenApplyAsync`，`thenAcceptAsync`，`thenRunAsync` 等
以及背后所涉及使用到的线程池。



