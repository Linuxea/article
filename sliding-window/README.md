## Background

在日常需求中，我们可能经常会遇到类似的需求：
- 在一个在线购物网站中，为了防止刷单行为，限制每个用户在一定时间内只能下单一定数量

- 在一个API服务中，为了保护后端服务不被过多请求拖垮，限制每个客户端每秒钟只能发起一定数量的请求

- 在一个网络游戏中，为了防止玩家利用自动化工具刷经验，限制每个玩家在一定时间内只能完成一定数量的任务


限流是一种流量控制策略，旨在控制进入系统或服务的请求数量，以保护系统在面对大量请求时仍能维持良好的性能和稳定性。

限流通常用于防止系统过载，降低延迟，避免资源耗尽，从而确保关键服务在高并发环境下正常运行。



## Introduction

限流可以抽象出一个 ”窗口“。
窗口限流的核心思想是在一个时间窗口内，对请求数量进行限制。这个时间窗口可以是固定的，也可以是随着时间推移而移动的。通过控制时间窗口内的请求量，可以有效地平衡系统负载，提高系统稳定性。

接下来，我们会从窗口固定与滑动的维度来进行分析与实现。



## 限流实现及其优缺点

我们使用 java 实现代码编程。
首先会定义好一个限流的接口，统一不同的实现方式。
```java
public interface RateLimiter {

  /**
   * 尝试获取令牌
   *
   * @return true:获取成功 false:获取失败
   */
  boolean tryAcquire();
}
```

### 固定窗口

#### 均匀平缓的固定窗口

当需要在整个时间段内平均分配请求时。
例如，在视频流传输、在线游戏等对实时性要求较高的场景，此方案可以有效平滑传输速率，避免突发流量对系统造成冲击。

将时间分成固定大小的窗口，每个窗口内允许`固定`数量的请求。


我们可以使用令牌桶（Token Bucket）算法。令牌桶算法允许我们在一段时间内限制请求数。每个请求需要消耗一个令牌。我们将桶中的令牌数量限制为Y，X单位时间内添加Y个令牌。当桶中没有令牌时，请求将被拒绝。
以下是使用Java实现的一个示例：

```java
/**
 * 固定时间间隔限流器
 */
public class FixedIntervalRateLimiter implements RateLimiter {

  private final int maxTokens;
  private final AtomicInteger tokens;

  public FixedIntervalRateLimiter(int maxTokens, long period, TimeUnit timeUnit,
      ScheduledExecutorService scheduler) {
    this.maxTokens = maxTokens;
    this.tokens = new AtomicInteger(maxTokens);
    scheduler.scheduleAtFixedRate(this::addToken, 0, period / maxTokens, timeUnit);
  }

  private void addToken() {
    if (tokens.get() < maxTokens) {
      tokens.incrementAndGet();
    }
  }

  public boolean tryAcquire() {
    return tokens.getAndDecrement() > 0;
  }
}
```

每隔一段固定的时间间隔（X单位时间 / Y次请求）增加一个令牌，如果我们设置maxTokens为5，period为1秒，那么每隔1/5(结果为1)秒就会触发一次增加令牌的操作。

优点：这种方案可以平滑地控制请求速率，避免在短时间内产生大量请求导致系统压力过大。


### 应对突发具有弹性的固定窗口

当允许在短时间内处理较多请求，同时在长时间尺度上限制请求速率时。
例如，在API调用、短时任务调度等场景中，此方案可以在保证长期稳定性的前提下，应对短时突发流量。

```java
package com.linuxea;

import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class FixWindowRateLimiter implements RateLimiter {

  private final int maxTokens;
  private final AtomicInteger tokens;

  public FixWindowRateLimiter(int maxTokens, long period, TimeUnit timeUnit,
      ScheduledExecutorService scheduler) {
    this.maxTokens = maxTokens;
    this.tokens = new AtomicInteger(maxTokens);
    scheduler.scheduleAtFixedRate(this::addTokens, period, period, timeUnit);
  }


  private void addTokens() {
    tokens.set(Math.min(tokens.get() + maxTokens, maxTokens));
  }

  public boolean tryAcquire() {
    return tokens.getAndDecrement() > 0;
  }
}

```

我们对 `FixedIntervalRateLimiter` 类进行了以下更改实现新的方式：

修改了addToken方法的名称为addTokens，并且将其更改为一次性增加Y个令牌。我们将当前令牌数量加上Y，并确保结果不超过最大令牌数量。

在构造函数中，我们将scheduleAtFixedRate方法的参数更改为this::addTokens，并将时间间隔设置为period。

现在，每隔X单位时间，addTokens方法会被执行一次，并且一次性增加Y个令牌。

优点：这种方案允许在短时间内处理突发请求，提供了一定程度的弹性。

### 时间滑动窗口





## 总结

