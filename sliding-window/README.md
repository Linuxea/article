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

- 当允许在短时间内处理较多请求，同时在长时间尺度上限制请求速率时。
例如，在API调用、短时任务调度等场景中，此方案可以在保证长期稳定性的前提下，应对短时突发流量。

- 系统容忍短时间内的流量波动：当系统可以承受短时间内的流量波动时。

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

优点：

- 突发处理能力：在每个时间窗口内，可以立即处理Y个请求，对于突发流量有较好的应对能力。

- 易于理解和实现：相对于平滑地分配请求，第二种方案更直观，实现起来也相对简单。

缺点：

- 窗口边界问题：在时间窗口的边界处，可能会出现连续处理2 * Y个请求的情况，从而导致短时间内的流量过大，对系统产生较大压力。例如，如果在一个时间窗口的末尾处理了Y个请求，在下一个时间窗口的开始又处理了Y个请求，这样就相当于在很短的时间内处理了2 * Y个请求。

- 对流量控制较为粗糙：相比于第一种方案，这种方案对流量的控制较为粗糙，不能平滑地控制请求速率。

### 时间滑动窗口


在需要平滑地控制请求数量，避免窗口边界问题的场景下，我们可以把它设计成一个非固定时间窗口，而是一个随着时间推移而移动的窗口。
这种算法允许在任意时间点开始的X单位时间内最多处理Y个请求。为了实现这个功能，我们需要记录请求的时间戳，并在判断请求是否允许时检查滑动窗口内的请求数量。


代码实现如下：
```java
package com.linuxea.impl;

import com.linuxea.RateLimiter;
import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import redis.clients.jedis.Jedis;

public class SlidingWindowRateLimiter implements RateLimiter {

  private final int maxTokens;
  private final Jedis jedis;
  private final Integer windowSize;
  private final TimeUnit widowSizeUnit;
  private final String key;

  public SlidingWindowRateLimiter(int maxTokens, Jedis jedis, TimeUnit widowSizeUnit,
      Integer windowSize, ScheduledExecutorService scheduler) {
    this.maxTokens = maxTokens;
    this.jedis = jedis;
    this.windowSize = windowSize;
    this.widowSizeUnit = widowSizeUnit;
    //identifier for the sorted set
    this.key = "sliding_window" + UUID.randomUUID();
    // Schedule a task to remove events older than the lower bound from the sorted set
    scheduler.scheduleAtFixedRate(this::removeOlderEventOutOfWindows, windowSize, windowSize,
        widowSizeUnit);
  }

  @Override
  public boolean tryAcquire() {
    long tsSeconds = Instant.now().getEpochSecond();

    // Add the event to the sorted set
    jedis.zadd(key, tsSeconds, UUID.randomUUID().toString());

    // Calculate the lower and upper bounds for the sliding window
    long lowerBound = tsSeconds - widowSizeUnit.toSeconds(windowSize);

    // Count the events in the sliding window
    long eventCount = jedis.zcount(key, String.valueOf(lowerBound + 1), String.valueOf(tsSeconds));

    return eventCount <= maxTokens;
  }

  private void removeOlderEventOutOfWindows() {
    long tsSeconds = Instant.now().getEpochSecond();
    long lowerBound = tsSeconds - widowSizeUnit.toSeconds(windowSize);
    // Remove events older than the lower bound from the sorted set
    jedis.zremrangeByScore(key, "-inf", String.valueOf(lowerBound));
  }

}
```

滑动窗口限流算法的分析：

优点：

平滑流量控制：滑动窗口算法能够在任意时间点开始的X单位时间内平滑地控制请求数量，避免了窗口边界问题。
弹性处理能力：在窗口内允许处理Y个请求，能够应对突发流量。
缺点：

实现复杂度较高：与固定窗口限流算法相比，滑动窗口限流算法需要记录请求的时间戳，实现起来相对复杂。

计算开销较大：需要维护请求时间戳列表，每次尝试获取令牌时，都需要遍历列表以移除窗口之外的请求，这可能导致较高的计算开销。


## 总结

以上三种限流方案各具特点，在实际应用中，需要根据系统的需求、特点和场景来选择合适的限流策略。
- 固定间隔增加令牌适用于需要平滑控制请求速率的场景，但对突发流量处理能力较弱；
- 固定窗口限流适用于突发处理能力要求较高的场景，但存在窗口边界问题；
- 滑动时间窗口限流在平滑流量控制和弹性处理能力方面具有优势，但实现复杂度和计算开销较高。

在选择限流方案时，需要根据具体情况权衡这些因素。