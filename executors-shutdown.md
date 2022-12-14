# ExecutorService - Waiting for Threads to Finish

## Overview

java 中众所周知的 `ExecutorService` 框架让并发处理任务变得轻而易举。

今天我们会举一些例子来演示如何优雅地关闭并等待 `ExecutorService` 中的线程完成它们的任务。

## After Executor's Shutdown

当使用 `Executor` 时，我们可以通用调用它的 `shutdown()` 或者 `shutdowNow()` 方法，然而它并不会等待所有的线程停止执行。

版本1
```java
package com.linuxea.javaconcurrent;

import java.util.Random;
import java.util.concurrent.*;

public class Main {

    public static void main(String[] args) {

        // 使用 Executors 创建一个固定线程数量为 1 的 executorService
        ExecutorService executorService = Executors.newFixedThreadPool(1);

        // 提交一个永远不可能完成的任务
        executorService.execute(() -> {
            try {
                TimeUnit.DAYS.sleep(Integer.MAX_VALUE);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });

        // 调用 executorService 的 shutdown
        executorService.shutdown();
    }

}

```

我们可以看到 `main` 线程迟迟无法结束。这并不是因为在等待线程池中的任务完成。

而是因为当前程序进程内仍然存在着非守护线程.

我们通过自定义 `ThreadFactory` 来创建非守护的线程。

版本2
```java
package com.linuxea.javaconcurrent;

import java.util.Random;
import java.util.concurrent.*;

public class Main {

    public static void main(String[] args) {

        // 使用 Executors 创建一个固定线程数量为 1 的 executorService
        ExecutorService executorService = Executors.newFixedThreadPool(1, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = Executors.defaultThreadFactory().newThread(r);
                thread.setDaemon(true); // 设置创建出来的线程为守护线程，当一个进程内存在着只有一个或多个守护线程时，整个进程就会退出
                return thread;
            }
        });

        // 提交一个永远不可能完成的任务
        executorService.execute(() -> {
            try {
                TimeUnit.DAYS.sleep(Integer.MAX_VALUE);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });

        // 调用 executorService 的 shutdown
        executorService.shutdown();
    }

}

```
```output
Process finished with exit code 0
```


这一次程序在任务没有完成的情况下正常退出了。
回到主题，那我们如何等待线程的正常结束再退出。

`shutdown` 无法保证这一点，我们可以使用另外一个方法 `awaitTermination`：

```java
    /**
     * Blocks until all tasks have completed execution after a shutdown
     * request, or the timeout occurs, or the current thread is
     * interrupted, whichever happens first.
     *
     * @param timeout the maximum time to wait
     * @param unit the time unit of the timeout argument
     * @return {@code true} if this executor terminated and
     *         {@code false} if the timeout elapsed before termination
     * @throws InterruptedException if interrupted while waiting
     */
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
```

版本3
```java
package com.linuxea.javaconcurrent;

import java.util.Random;
import java.util.concurrent.*;

public class Main {

    public static void main(String[] args) {

        // 使用 Executors 创建一个固定线程数量为 1 的 executorService
        ExecutorService executorService = Executors.newFixedThreadPool(1, runnable -> {
            Thread thread = Executors.defaultThreadFactory().newThread(runnable);
            thread.setDaemon(true); // 设置创建出来的线程为守护线程，当一个进程内存在着只有一个或多个守护线程时，整个进程就会退出
            return thread;
        });

        executorService.execute(() -> {
            try {
                TimeUnit.SECONDS.sleep(10L);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("task finish");
        });

        // 调用 executorService 的 shutdown
        executorService.shutdown();

        try {
            // 在调用 shutdown 后
            // 使用 awaitTermination 来阻塞所有 executorService 线程
            // 1.直到所有的任务完成
            // 2.或者达到设定的一个超时时间
            // 3.或者收到线程中断异常
            // 满足第一个触发的以上三个条件之一
            boolean awaitTermination = executorService.awaitTermination(15L, TimeUnit.SECONDS);
            System.out.println(awaitTermination);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

}

```


##  Using CountDownLatch


使用 `CountDownLatch` 来通知任务的完成。

```java
package com.linuxea.javaconcurrent;

import java.util.Random;
import java.util.concurrent.*;
import java.util.function.Consumer;
import java.util.function.IntConsumer;
import java.util.stream.IntStream;

public class Main {

    public static void main(String[] args) {

        // 使用 Executors 创建一个固定线程数量为 50 的 executorService
        ExecutorService executorService = Executors.newFixedThreadPool(50, runnable -> {
            Thread thread = Executors.defaultThreadFactory().newThread(runnable);
            thread.setDaemon(true); // 设置创建出来的线程为守护线程，当一个进程内存在着只有一个或多个守护线程时，整个进程就会退出
            return thread;
        });

        // 初始化 100 个在所有调用 await() 方法的线程之前它可以递减的次数
        CountDownLatch countDownLatch = new CountDownLatch(100);

        IntConsumer executes = integer -> executorService.execute(() -> {
            try {
                TimeUnit.SECONDS.sleep(3L);
                System.out.println("我完成了" + integer);

                // 标识为完成
                countDownLatch.countDown();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        IntStream.range(0, 100).forEach(executes);


        // await all cuntDownLatch to zero
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        // it does not matter;
//        executorService.shutdown();
    }

}


```


## Using invokeAll()


`invokeAll()` 方法执行了给定的任务集合。
并在所有任务完成时返回 `Future` 列表，`Future` 对象保存了任务的状态与结果。

通用调用列表中每个元素的 `Future.isDone` 都会得到 `true` 的结果。


```java
package com.linuxea.javaconcurrent;

import java.util.List;
import java.util.Random;
import java.util.concurrent.*;
import java.util.function.Consumer;
import java.util.function.IntConsumer;
import java.util.stream.Collectors;
import java.util.stream.IntStream;

public class Main {

    public static void main(String[] args) {

        // 使用 Executors 创建一个固定线程数量的executorService
        ExecutorService executorService = Executors.newFixedThreadPool(50, runnable -> {
            Thread thread = Executors.defaultThreadFactory().newThread(runnable);
            thread.setDaemon(true); // 设置创建出来的线程为守护线程，当一个进程内存在着只有一个或多个守护线程时，整个进程就会退出
            return thread;
        });

        List<CallInt> callables = IntStream.range(0, 100).mapToObj(i -> new CallInt()).collect(Collectors.toList());
        try {
            // 批量的形式执行任务
            // 所有的任务完成时会返回一个 Future 的列表
            List<Future<Integer>> futures = executorService.invokeAll(callables);

            // 断言大小为给定参数集合大小
            assert futures.size() == 100;
            // 获取任务执行结果
            futures.stream().map(it -> {
                try {
                    return it.get();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                } catch (ExecutionException e) {
                    throw new RuntimeException(e);
                }
            }).forEach(System.out::println);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }



        // it does not matter;
//        executorService.shutdown();
    }

    public static class CallInt implements Callable<Integer> {

        @Override
        public Integer call() throws Exception {
            TimeUnit.SECONDS.sleep(5L);
            return new Random().nextInt(1000);
        }
    }

}

```


## Using ExecutorCompletionService

另外一种运行多线程的方式是使用 `ExecutorCompletionService`.

`ExecutorCompletionService` 使用了一个给定的 `ExecutorService` 来执行任务。

它与 `invokeAll()` 的一个不同之处在于代表返回任务执行的 `Futures` 的顺序。

`ExecutorCompletionService` 使用了一个队列结构来按完成顺序存储 `Future` 结果，

然而 `invokeAll()` 是按原顺序返回。

```java
package com.linuxea.javaconcurrent;

import java.util.List;
import java.util.Random;
import java.util.concurrent.*;
import java.util.stream.Collectors;
import java.util.stream.IntStream;

public class Main {

    public static void main(String[] args) {

        // 使用 Executors 创建一个固定线程数量的executorService
        ExecutorService executorService = Executors.newFixedThreadPool(50, runnable -> {
            Thread thread = Executors.defaultThreadFactory().newThread(runnable);
            thread.setDaemon(true); // 设置创建出来的线程为守护线程，当一个进程内存在着只有一个或多个守护线程时，整个进程就会退出
            return thread;
        });

        ExecutorCompletionService<Integer> completionService = new ExecutorCompletionService<>(executorService);


        List<Future<Integer>> futures = IntStream.range(0, 2).mapToObj(CallInt::new).map(completionService::submit).collect(Collectors.toList());

        // 预期按任务完成时间打印 0 1
        futures.stream().map(it -> {
            try {
                return it.get();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } catch (ExecutionException e) {
                throw new RuntimeException(e);
            }
        }).forEach(System.out::println);



        // it does not matter;
//        executorService.shutdown();
    }

    public static class CallInt implements Callable<Integer> {

        private final Integer sleep;

        public CallInt(Integer sleep) {
            this.sleep = sleep;
        }


        @Override
        public Integer call() throws Exception {
            TimeUnit.SECONDS.sleep(sleep);
            return sleep;
        }
    }

}

```


## Conclusion

可以看到取决于不同的场景，我们能使用不同的方式来达到等待线程完成任务的目的。

## 参考

- [1] [ExecutorService – Waiting for Threads to Finish](https://www.baeldung.com/java-executor-wait-for-threads)
- [2] [shutdown in ExecutorService](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/ExecutorService.html#shutdown())
- [3] [invokeAll in ExecutorService](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/ExecutorService.html#invokeAll(java.util.Collection))
- [4] [ExecutorCompletionService](https://docs.oracle.com/en/java/javase/12/docs/api/java.base/java/util/concurrent/ExecutorCompletionService.html#%3Cinit%3E(java.util.concurrent.Executor))