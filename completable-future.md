<!-- # Introduction In CompletableFuture -->

> Java 1.8 带来了如 `Lambda` 表达式，`Stream` 等丰富的功能，除此之外，还有  `CompletableFuture`


# What’s a CompletableFuture?

`CompletableFuture` 是 `Java` 的异步编程框架。

> 异步编程是一种通过在与主应用程序线程不同的线程上运行任务并通知主线程其进度、完成或失败来编写非阻塞代码的方法。
主线程不会阻塞/等待任务完成，它可以并行执行其他任务。

异步编程的这种并行性可以极大地提高程序的性能。

## Limitations of Future

`jdk1.5` 引入的 `Future` 代表异步计算的结果。

`CompletableFuture` 相比 `Future`，在异步编程体验与功能上丰富度上都更进一步。
- 支持手动完成任务
- 非阻塞式编程方式继续下一个任务
- 多个任务支持链接串联
- 支持多个任务整合
- 支持异常处理



# Creating a CompletableFuture

使用空构造器创建一个任务返回结果值类型为 `String` 的最简单的 `CompletableFuture`

```java
CompletableFuture<String> completableFuture = new CompletableFuture<>();
```

设置超时返回默认值的形式来终止任务
```java
completableFuture.completeOnTimeout("default", 3, TimeUnit.SECONDS);
```

`join` 会阻塞直到任务完成。


>note: get 也能够实现 join 相同的功能。但是 get 会抛出受检异常，需要开发者进行捕获

```java
completableFuture.join();
```


# Running asynchronous computation using runAsync()

`runAsync()` 是以异步方式运行 `Runnable`.

>Note: `Runnable` 是最常用的函数式接口之一，表明一个不需要参数也没有返回的任务

```java
CompletableFuture.runAsync(() -> System.out.println("I am done")).join();
```


# Run a task asynchronously and return the result using supplyAsync()


相比 `runAsync()` ，如果想从异步任务中获取一个结果，`supplyAsync()` 是你的选择。
参数是 `Supplier`。


>Note:  `Supplier<T>` 是最常用的函数式接口之一， 用来提供使用者一个类型为 `T` 的值

```java
CompletableFuture.supplyAsync(() -> 1);
```



# Transforming and acting on a CompletableFuture

直接使用 `get` 或者 `join` 会阻塞当前线程直到任务完成。

但这是我们想要的吗。

对此，`CompletableFuture` 提供了回调机制来完善异步系统。

我们不需要等待任务完成，只需要在回调函数中完成编写任务结果的处理逻辑。

你可以使用 `thenApply()`、`thenAccept()` 和 `thenRun()` 方法将回调方法附加到 `CompletableFuture`


## thenApply()

使用 `thenApply()` 处理任务结果并进行转化。

它使用了 `Function<T,R>` 作为参数。

>Note: `Function<T,R>` 也是最常用的函数式接口之一，用来对类型为 T 的输入参数进行转换为类型为 R 的输出结果

```java
// 将任务结果 linuxea 转化为对其求字符串长度
CompletableFuture.supplyAsync(() -> "linuxea").thenApply(String::length).join()
```

可以通过附加多个 `thenApply()` 方法来编写一系列转换。

上一个 `thenApply()` 的结果会传递给下一个 `thenApply()`。

```java
CompletableFuture.supplyAsync(() -> "linuxea") // 任务结果返回 linuxea 字符串
        .thenApply(String::length) // 对任务结果取字符串长度
        .thenApply(length -> length > 5) // 判断长度是否大于 5
        .thenApply(String::valueOf) // 将 Boolean 转化成字符串形式
        .join();
```

# thenAccept() and thenRun()

与 `thenApply()` 相比，如果你不想从回调中返回任务结果，而只是想运行借由上一个任务的结果完成一些代码逻辑，`thenAccept()` 是你的选择。

`thenAccept()` 参数是一个 `Consumer<T>`。

>Note: `Consumer<T>` 也是最常用的函数式接口之一，对类型为 T 的参数进行消费且不返回任何结果值

```java
CompletableFuture.supplyAsync(() -> "linuxea") // 任务结果返回 linuxea 字符串
        .thenAccept(System.out::println) // 直接把任务结果进行打印
    ;
```

与 `thenAccept()` 相似的是 `thenRun()` 。

`thenRun()` 的参数是 `Runnable`。它不需要上一个任务的结果，也不返回任何结果。

当只需要通知某些人任务已经完成而不需要具体的任务结果时，它会派上用场的。

```java
CompletableFuture.supplyAsync(() -> "linuxea") // 任务结果返回 linuxea 字符串
        .thenRun(() -> System.out.println("I am done")) // 通知任务完成
```


>Note: `Runnable` 也是我们最熟悉的函数式接口之一。它没有参数也没有返回值


# Combining two CompletableFutures together

## thenCompose()

将有依赖关系的任务通过 `thenCompose()` 组合起来

```java
// 根据 uid 获取用户名
public CompletableFuture<String> getUserName(Long uid) {
    //忽略参数直接返回用户名
    return CompletableFuture.supplyAsync(() -> "linuxea");
}

// 根据用户名获取 id card
public CompletableFuture<String> getIDCard(String userName) {
    //忽略参数直接返回 id card
    return CompletableFuture.supplyAsync(() -> "1234567890");
}
```
以上两个任务能够发现是具有依赖关系的。

我们需要根据 `uid` 完成获取用户名的任务，再把任务结果传递到获取 idCard 的任务去。

使用 `thenCompose(CompletableFuture)` 组合两个 `CompletableFuture`。
```java
getUserName(uid).thenCompose(this::getIDCard).join()
```

如果使用上面提到的 `thenApply()` 会有什么不同吗。同样是把一个结果映射为另外一个结果。

```java
CompletableFuture<CompletableFuture<String>> completableFutureCompletableFuture = this.getUserName(uid).thenApply(this::getIDCard);
```

我们返回的是一个嵌入 `CompletableFuture` 的 `CompletableFuture<CompletableFuture<String>>`。
如果我们想要一个扁平化的 `CompletableFuture` 那么使用  `thenCompose`。

## thenCombine()

`thenCompose()` 整合了具有依赖关系的 `CompletableFuture`。

`thenCombine()` 是将两个独立的 `CompletableFuture` 并行处理，并在各自任务完成时进行整合。

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


# Combining multiple CompletableFutures together

我们使用 `thenComponse()` 与 `thenCombine()` 来合并两个 `CompletableFutures`。

如果我们想要合并任意数量的 `CompletableFutures` 呢？

好吧。你可以使用如下方法来实现：

```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
```

###  allOf

`allOf` 使用场景是在并行完成一系列独立的 `CompletableFuture` 后，可以实现后续处理。

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


## anyOf

`CompletableFuture.anyOf()` 如字面上意思，在给定的多个 `CompletableFutures` 中任一完成时返回一个同样类型结果的新的 `CompletableFuture`。


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


# CompletableFuture Exception Handling

我们已经探索了如何创建 `CompletableFuture`, 以及转化结果，合并多个 `CompletableFutures` 的方法。

现在我们来了解下当出现异常时该做什么。

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

创建一个 `Future` 时，如果异常发生在一开始的 `supplyAsync()` 阶段，那没有任何一个 `thenApply()` 回调逻辑被调用，同时 `future` 将面临处理这个异常。

如果异常出现在第一个 `thenApply()`, 那么第二跟第三个 `thenApply()` 将不会触发回调，同样 `future` 将面临处理这个异常。

以此类推。


## 1. Handle exceptions using exceptionally() callback


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

`exceptionally` 在上一步给定的函数触发异常时返回定义新的 `CompletableFuture`。
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

## 2. Handle exceptions using the consumer whenComplete method

对异常的具体处理可以使用 `whenComplete()`, 它的参数是 `BiConsumer`。

通过上一步的结果与异常实现相应的消费逻辑。但是它不具备更改结果值的 `handle` 能力。

```java
Supplier<String > supplier = () -> { throw new RuntimeException("error is me");};
CompletableFuture.supplyAsync(supplier)
    .whenComplete((s, throwable) -> {
        System.out.println("has a error ?:" + (throwable != null));
        System.out.println("future result is " + s);
    })
    .exceptionally(throwable -> "default");
```


## 3. Handle exceptions using the generic handle() method

相比 `whenComplete`，`handle()` 是对付异常更加通用的处理方式。

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



# Conclusion


在上述示例中，介绍了 `CompletableFuture` 部分重要且有用的概念。
还有部分异步 `API` 没有介绍到，比如转换时以异步方式处理的 `thenApplyAsync`，`thenAcceptAsync`，`thenRunAsync` 以及背后所涉及使用到的线程池。



## 参考

- [1] [Java CompletableFuture Tutorial with Examples](https://www.callicoder.com/java-8-completablefuture-tutorial/#combining-multiple-completablefutures-together)
- [2] [CompletableFuture<T> class: join() vs get()](https://stackoverflow.com/questions/45490316/completablefuturet-class-join-vs-get)
- [3] [CompletableFuture in Java Simplified](https://medium.com/javarevisited/completablefuture-usage-and-best-practises-4285c4ceaad4)
- [4] [异步编程百度百科](https://baike.baidu.com/item/%E5%BC%82%E6%AD%A5%E7%BC%96%E7%A8%8B/1349830?fr=aladdin)


