# How To Terminate Threads in Java

> I had this epiphany about how a thread is interrupted in Java recently. I’ve never read up on it and just got some piecemeal information from stackoverflow et al. when looking to how to deal with the dreaded InterruptedException being thrown out of a Thread.sleep().

> 最近开始对线程的中断有了新的看法，如之前未关注过的不知道何处抛出的 InterruptedException


## How Java Threads Terminate

In Java, a thread terminates when the top-level method, i.e., the one that implements Runnable.run() or Callable.call() naturally exits.

在 `java` 中，实现了 `Runnable.run()` 或者 `Callable.call()` 类的顶层方法退出，意味着线程的正常退出。

There is a Thread.stop(), but it just kills the thread and may leave your application in an inconsistent state. In other words, you only want to call it when you want to kill your application anyway, and this can be done easier with just calling System.exit().

存在着一个方法 `Thread.stop()` ，但是它仅仅杀死线程，这很有可能让你的应用处于一个数据不一致的状态。
换句话讲，你只会在无视后果时都要杀死这个线程时调用它。在这种情况下，`System.exit()` 也许是个更简单的选项。

There are cases when you need to terminate a thread sooner than it comes to a natural end. One example is when your thread is designed to run forever because it processes some sort of events, polls the database or watches files. For example:


有几种场景需要你尽可能快地关闭线程来达到线程正常结束生命周期。

一个场景是你设计的永久运行的线程中(这样的线程通常是永远不会停止的除非进行干预)，用来处理某些种类事件，轮询数据库或者监控文件。比如：
```java
new Thread(() -> {
           for(;;) {
               Event event = getEvent();
               if(event != null) {
                   process(event);
               }
           }
        }).start();
```

Another case is long-running threads where you simply cannot afford to wait until they end. On the desktop, a user may have clicked on a “quit” or “cancel” button and expects the operation to end in a reasonable time. 


另一个场景是用户点击关闭桌面应用并期望在一个合理的时间内退出应用。
否则我们需要终止一个无法按预期终止的线程。

On the server-side, operating systems grant processes only a limited time to shut down gracefully before they force-kill them.


第三个场景，服务端在强制杀死进程时，会允许进程在有限时间内优雅关闭。


## Interrupting Threads

The Java-native way to stop threads it the Thread.interrupt() method. Note that the JVM does not in any way automatically interrupt threads. You have to implement the Thread.interrupt() call yourself, depending on your use case. You typically do this in a shutdown hook. Spring Boot comes with a default shutdown hook that looks for methods annotated with @PreDestroy, where you can put the interruption logic.

`Thread.interrupt()` 是 `java` 原生停止线程的方法。需要注意的是 `JVM` 在任何情况下都不会自动打断线程。你必须要自己根据场景实现 `Thread.interrupt()` 的调用处理。通常来讲一般是在 `shutdown hook` 处理。

By itself, Thread.interrupt() does not do much because Java has no idea how to force a thread to terminate. It is the responsibility of the developer to check with the static method Thread.interrupted() if the current thread has been interrupted. If this method returns true, you should terminate the thread as quickly as possible and only execute the minimum necessary clean-up code.

`Thread.interrupt()` 本身并不会做太多事情, 这是因为 `java` 并不知道如何强制终止一个线程。

从开发者角度来讲，需要自己通过 `Thread.interrupted()` 来检查当前线程是否被中断。

如果这个方法返回 `true`，你应该尽可能快地以执行最小必要的清理代码，并且停止线程。


```java
import java.time.LocalDateTime;

public class Main {

    public static void main(String[] args) {

        Thread thread = new Thread(() -> {
            while (!Thread.interrupted()) {
                String log = String.format("hello %d", LocalDateTime.now().getSecond());
                System.out.println(log);
            }
            System.out.println("end normally");
        });

        thread.start();

        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("I jvm am done");
            thread.interrupt();
            try {
                thread.join();
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }));

    }
}
```

Thread.interrupted() does reset the thread’s state, means it is no longer in interrupted state. Hence, code like below would not work:

需要注意的是，`Thread.interrupted()` 会重置线程状态，这意味着线程将不再处于 `interrupted` 状态。
因此如下的代码将不会正常工作。

```java
Thread thread = new Thread(() -> {

    while (true) {
        if (Thread.interrupted()) {
            System.out.println("interrupted one");
        }

        if (Thread.interrupted()) {
            System.out.println("interrupted two and will never occurs");
        }
    }

});
```

If the first check returns true, the second one would return false and processEventPart2() is executed. That is why it is generally recommended to interrupt the current thread again after calling Thread.interrupted():

如果第一个检查返回 `true`, 那么第二个检查将会是 `false`, 因此第二句将永远不会被执行。
这也是为什么通常会建议在调用 `Thread.interrupted()` 后再次打断当前线程。

```java
Thread thread = new Thread(() -> {

    while (true) {
        if (Thread.interrupted()) {
            System.out.println("interrupted one");
            Thread.currentThread().interrupt();
        }

        if (Thread.interrupted()) {
            System.out.println("interrupted two and will occurs");
            Thread.currentThread().interrupt();
        }
    }

});
```

This ensures that subsequent checks still see an interrupted thread.

这能确保随后的检查依然能够看到线程打断中的状态。

> 除了 Thread.interrupted()， 也可以使用 Thread.currentThread().isInterrupted() 来判断当前线程是否被打断。调用它不会对线程是否打断的状态产生副作用




## Better Do Your Own Thing

When you can’t use Thread.interrupt(), you need to implement your own logic. Here’s an example for stopping an endless loop:

当不能使用 `Thread.interrupt()` 触发中断时，开发者需要自己实现逻辑处理。
举个例子来终止一个无限循环：
```java
import java.util.concurrent.atomic.AtomicBoolean;



@FunctionalInterface
public interface InfiniteRunnable extends Runnable {

    /**
     * running flag
     */
    AtomicBoolean running = new AtomicBoolean(true);

    void process();


    default void run() {
        while (running.get() && !Thread.currentThread().isInterrupted()) {
            this.process();
        }
        System.out.println("infinite run break");
    }

    default void stop(){
        this.running.set(false);
    }

}

```


AtomicBoolean is a JDK class that implements a thread-safe boolean variable. The stop() method simply sets it to false, which exists the loop the next time while is executed. You lose the ability to stop individual threads, but you don’t need that for a graceful shutdown.

`AtomicBoolean` 是 `JDK` 实现了线程安全的 `Boolean` 变量的一个类。
`InfiniteRunnable.stop()` 方法简单地把它的值设置为 `false`, `run` 方法中下一次执行时将会因此而退出循环。

你不再需要有针对线程的停止能力，但同时也不需要它来实现优雅关闭。


## Stop Sleeping and Waiting

在没有 `Thread.interrupt()` 的情况下，`sleep()`, `wait()` 不能响应中断从而会阻塞直到超时时间。
这里有三种方式来处理这种情况：
- 如果 `sleep` 或者 `wait` 时间很短，而你有30秒的时间来停止整个进程。那几秒的阻塞时间将不会是大问题。
- 如果 `sleep` 时间更长，那么使用 `scheduled` 线程替代会是一个更好的选择。
- 将一个长时间的 `sleep` 拆分成多个小的 `sleep`，在它们中间检查 `stop` 标识是否已经是 `true`




## Graceful shutdown in Spring Boot

One of the many advantages in using Spring Boot is that its components are configured to gracefully shut down. However, if you start your own threads, you have to take care of them during shutdown as well. A good way is using ThreadPoolTaskExceutor, even if you don’t need the thread pooling functionality. Just make sure to size the pool correctly to avoid threads queuing up or wasting resources.


使用 `Spring Boot` 的众多好处之一就是它的组件都已经配置为优雅关闭。

不过，如果你启用的是自己的线程，你还是需要在关闭期间关注它们。

一个好的实践是使用 `ThreadPoolTaskExceutor`, 即使在你不需要线程池功能的情况下，只需要确保线程池大小的正确配置，来避免资源浪费。

Spring Boot calls shutdown() on ThreadPooolTaskExceutor when the application terminates. By default, it then interrupts the threads. If you don’t want that for the reasons listed above, then you need to set setWaitForTasksToCompleteOnShutdown() to true. Another useful feature is setting a delay after the termination with setAwaitTerminationSeconds() or setAwaitTerminationMillis(). This causes Spring Boot to suspend its shutdown sequence until all threads have terminated or the specified timeout occurs. Without this delay, certain resources like database pools may be shutting down while a thread that uses them is still running.

`Spring Boot` 会在应用终止时调用 `ThreadPooolTaskExceutor` 的 `shutdown()`。

默认地，它会在之后调用线程的 `Thread.interrupt()`。

如果因为上述原因你不想要发生打断线程，你需要设置 `setWaitForTasksToCompleteOnShutdown()` 为 `true`

另外一个有用的功能是使用 `setAwaitTerminationSeconds` 或者 `setAwaitTerminationMillis()` 设置一个延迟时间，
这会导致 `Spring Boot` 挂起其关闭逻辑，直到所有线程都已终止或超时。

如果没有这种延迟，某些资源（如数据库池）可能会在使用它们的线程仍在运行时关闭。


如果你想使用上述例子自定义的线程终止方法，可以在 `Spring Boot` 配置中继承 `ThreadPooolTaskExceutor` 并覆盖 `shutdown()` ，如下所示：

```java
@Bean
public AsyncTaskExecutor asyncTaskExecutor(InfiniteRunnable infiniteRunnable) {
    ThreadPoolTaskExecutor te = new ThreadPoolTaskExecutor() {
        @Override
        public void shutdown() {
            // invoke shutdown
            infiniteRunnable.stop();
            super.shutdown();
        }
    };
    te.setCorePoolSize(1);
    te.setMaxPoolSize(1);
    te.setWaitForTasksToCompleteOnShutdown(true);
    te.setAwaitTerminationSeconds(3);
    return te;
}
```