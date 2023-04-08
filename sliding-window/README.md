## Background

在日常需求中，我们可能经常会遇到类似的需求：
- 在一个在线购物网站中，为了防止刷单行为，限制每个用户在一定时间内只能下单一定数量

- 在一个API服务中，为了保护后端服务不被过多请求拖垮，限制每个客户端每秒钟只能发起一定数量的请求

- 在一个网络游戏中，为了防止玩家利用自动化工具刷经验，限制每个玩家一个自然日能签到一次


限流是一种流量控制策略，旨在控制进入系统或服务的请求数量，防止系统过载，降低延迟，避免资源耗尽。以保护系统在面对大量请求时仍能维持良好的性能和稳定性。


## Introduction

限流可以抽象出一个 ”窗口“，窗口限流的核心思想是在一个时间窗口内，对请求数量进行限制。这个时间窗口可以是固定的，也可以是随着时间推移而移动的。我们从两个维度分析窗口：
- 可用大小：通过控制窗口内的请求量，可以有效地平衡系统负载，提高系统稳定性。
- 是否随时间移动：实现动态时间范围的流量控制，不需要进行窗口切换及应对随之产生的问题。


接下来，我们会以窗口角度进行分析与实现。



## Implement

我们使用 `java` 代码编程。
首先会定义好一个限流的接口，统一不同的实现方式。
```java
package com.linuxea;

/**
 * 限流器
 * <p> 统一限流接口
 * Created by Linuxea on 2019-04-28 22:10
 */
public interface RateLimiter {


  /**
   * 尝试获取令牌
   *
   * @return true:获取成功 false:获取失败
   */
  boolean tryAcquire();

  /**
   * 获取下一个窗口开始时间
   *
   * @return 下一个窗口开始时间
   */
  Long getNextWindowStartTimestamp();
}
```

```java
package com.linuxea.impl;

import com.linuxea.RateLimiter;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public abstract class AbsRateLimiter implements RateLimiter {

  protected int maxTokens;
  protected AtomicInteger tokens;
  protected long windowSize;
  protected TimeUnit windowTimeUnit;
  protected long delayTimeStamp;


  /**
   * 下一个窗口开始时间
   *
   * @return 下一个窗口开始时间
   */
  @Override
  public Long getNextWindowStartTimestamp() {
    return System.currentTimeMillis() + delayTimeStamp;
  }

}
```

### 固定窗口



#### 均匀平缓的固定窗口

使用背景:

当需要在整个时间段内平均分配请求时。
例如，在 web 等对实时性要求较高的场景，此方案可以有效平滑传输速率，避免突发流量对系统造成冲击。

将时间分成固定大小的窗口，每个窗口内允许`固定`数量的请求。

我们可以使用`令牌桶（Token Bucket）`算法。令牌桶算法允许我们在一段时间内限制请求数。每个请求需要消耗一个令牌。我们将桶中的令牌数量限制为Y，X单位时间内添加Y个令牌。当桶中没有令牌时，请求将被拒绝。
以下是使用Java实现的一个示例：

```java
package com.linuxea.impl;

import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * 固定时间间隔限流器
 */
public class FixedIntervalRateLimiter extends AbsRateLimiter {

  public FixedIntervalRateLimiter(int maxTokens, long windowSize, TimeUnit windowTimeUnit,
      ScheduledExecutorService scheduler) {
    this.maxTokens = maxTokens;
    this.tokens = new AtomicInteger(0);
    this.windowSize = windowSize;
    this.windowTimeUnit = windowTimeUnit;
    long windowSizeInMillis = windowTimeUnit.toMillis(windowSize);
    long intervalInMillis = windowSizeInMillis / maxTokens;
    this.delayTimeStamp = intervalInMillis;
    scheduler.scheduleAtFixedRate(this::addToken, intervalInMillis, intervalInMillis,
        TimeUnit.MILLISECONDS);
  }

  private void addToken() {
    int currentTokens;
    do {
      currentTokens = tokens.get();
      if (currentTokens >= maxTokens) {
        return;
      }
    } while (!tokens.compareAndSet(currentTokens, currentTokens + 1));
  }


  @Override
  public boolean tryAcquire() {
    int currentTokens = tokens.getAndDecrement();
    if (currentTokens > 0) {
      return true;
    } else {
      // 如果 tokens 变为负数，将其值恢复为0
      tokens.incrementAndGet();
      return false;
    }
  }

}
```

每隔一段固定的时间间隔（X单位时间 / Y次请求）增加一个令牌，如果我们设置maxTokens为50，period为1秒，那么每隔1/5秒就会触发一次增加令牌的操作。

优点：这种方案可以平滑地控制请求速率，避免在短时间内产生大量请求导致系统压力过大。


### 应对突发流量与具有弹性的固定窗口

使用背景：

- 当允许在短时间内处理较多请求，同时在长时间尺度上限制请求速率时。
例如，在API调用、短时任务调度等场景中，此方案可以在保证长期稳定性的前提下，应对短时突发流量。

- 系统容忍短时间内的流量波动：当系统可以承受短时间内的流量波动时。

```java
package com.linuxea.impl;

import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

public class FixWindowRateLimiter extends AbsRateLimiter {

  public FixWindowRateLimiter(int maxTokens, long windowSize, Long windowStart,
      TimeUnit windowTimeUnit, ScheduledExecutorService scheduler) {
    this.maxTokens = maxTokens;
    this.tokens = new AtomicInteger(System.currentTimeMillis() > windowStart ? maxTokens : 0);
    this.windowSize = windowSize;
    this.windowTimeUnit = windowTimeUnit;
    long windowSizeInMillis = windowTimeUnit.toMillis(windowSize);
    this.delayTimeStamp = calculateDelay(windowStart, windowSizeInMillis);
    scheduler.scheduleAtFixedRate(this::addTokens, delayTimeStamp, windowSizeInMillis,
        TimeUnit.MILLISECONDS);
  }

  public FixWindowRateLimiter(int maxTokens, long windowSize, TimeUnit windowTimeUnit,
      ScheduledExecutorService scheduler) {
    this.maxTokens = maxTokens;
    this.tokens = new AtomicInteger(0);
    this.windowSize = windowSize;
    this.windowTimeUnit = windowTimeUnit;
    long windowSizeInMillis = windowTimeUnit.toMillis(windowSize);
    scheduler.scheduleAtFixedRate(this::addTokens, 0, windowSizeInMillis, TimeUnit.MILLISECONDS);
  }

  /**
   * 计算延迟时间
   *
   * @param windowStartMillis 从什么时候开始计算
   * @return 延迟时间
   */
  private long calculateDelay(Long windowStartMillis, long windowSizeInMillis) {
    long ctMillis = System.currentTimeMillis();
    if (windowStartMillis == null) {
      return 0;
    } else if (windowStartMillis <= ctMillis) {
      //计算从上一个窗口开始到当前时间的已过去的毫秒数。
      long elapsedTimeSinceWindowStart = (ctMillis - windowStartMillis) % windowSizeInMillis;
      //计算上一个窗口还需要执行的时间
      return windowSizeInMillis - elapsedTimeSinceWindowStart;
    } else {
      return windowStartMillis - ctMillis;
    }
  }


  private void addTokens() {
    tokens.set(Math.min(tokens.get() + maxTokens, maxTokens));
  }

  public boolean tryAcquire() {
    int currentTokens = tokens.getAndDecrement();
    if (currentTokens > 0) {
      return true;
    } else {
      // 如果 tokens 变为负数，将其值恢复为0
      tokens.incrementAndGet();
      return false;
    }
  }

}
```

我们对 `FixedIntervalRateLimiter` 类进行了以下更改实现新的 `FixWindowRateLimiter`：

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

使用背景：

在需要平滑地控制请求数量，避免窗口边界问题的场景下，我们可以把它设计成一个非固定时间窗口，而是一个随着时间推移而移动的窗口。

这种算法允许在任意时间点开始的X单位时间内最多处理Y个请求。为了实现这个功能，我们需要记录请求的时间戳，并在判断请求是否允许时检查滑动窗口内的请求数量。


这里我们借助了 `redis` 的有序集合工具，代码实现如下：
```java
package com.linuxea.impl;

import java.time.Instant;
import java.util.UUID;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import redis.clients.jedis.Jedis;

public class SlidingWindowRateLimiter extends AbsRateLimiter {

  private final Jedis jedis;
  private final String key;

  public SlidingWindowRateLimiter(int maxTokens, Jedis jedis, TimeUnit windowTimeUnit,
      Integer windowSize, ScheduledExecutorService scheduler) {
    this.maxTokens = maxTokens;
    this.jedis = jedis;
    this.windowSize = windowSize;
    this.windowTimeUnit = windowTimeUnit;
    //identifier for the sorted set
    this.key = "sliding_window" + UUID.randomUUID();
    // Schedule a task to remove events older than the lower bound from the sorted set
    long windowSizeInMillis = windowTimeUnit.toMillis(windowSize);
    this.delayTimeStamp = windowSizeInMillis;
    scheduler.scheduleAtFixedRate(this::removeOlderEventOutOfWindows, windowSizeInMillis,
        windowSizeInMillis, TimeUnit.MILLISECONDS);
  }

  @Override
  public boolean tryAcquire() {
    long tsSeconds = Instant.now().getEpochSecond();

    // Add the event to the sorted set
    jedis.zadd(key, tsSeconds, UUID.randomUUID().toString());

    // Calculate the lower and upper bounds for the sliding window
    long lowerBound = tsSeconds - windowTimeUnit.toSeconds(windowSize);

    // Count the events in the sliding window
    long eventCount = jedis.zcount(key, String.valueOf(lowerBound + 1), String.valueOf(tsSeconds));

    return eventCount <= maxTokens;
  }

  private void removeOlderEventOutOfWindows() {
    long tsSeconds = Instant.now().getEpochSecond();
    long lowerBound = tsSeconds - windowTimeUnit.toSeconds(windowSize);
    // Remove events older than the lower bound from the sorted set
    jedis.zremrangeByScore(key, "-inf", String.valueOf(lowerBound));
  }

}
```

优点：

- 平滑流量控制：滑动窗口算法能够在任意时间点开始的X单位时间内平滑地控制请求数量，避免了窗口边界问题。
- 增强弹性处理能力：在窗口内允许处理Y个请求，能够应对突发流量。

缺点：

- 实现复杂度较高：与固定窗口限流算法相比，滑动窗口限流算法需要记录请求的时间戳，实现起来相对复杂。
- 计算开销较大：需要维护请求时间戳列表，每次尝试获取令牌时，都需要遍历列表以移除窗口之外的请求，这可能导致较高的计算开销。


## 举粟

回到开文讲到的几个例子，接下来通过在几个场景运用不同的限流来加深对代码的理解

### 在一个在线购物网站中，为了防止刷单行为，限制每个用户在30分钟内只能下单数量为1件
```java
package com.linuxea.impl;

import static org.junit.jupiter.api.Assertions.assertFalse;

import com.linuxea.RateLimiter;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;

/**
 * 在一个在线购物网站中，为了防止刷单行为，限制每个用户在30分钟内只能下单数量为1件
 */
public class ShoppingRateLimiter {

  @Test
  public void test() throws InterruptedException {
    ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();
    int maxShoppTimes = 1;
    int windowSize = 30;
    RateLimiter rateLimiter = new FixWindowRateLimiter(maxShoppTimes, windowSize,
        TimeUnit.MINUTES,
        scheduledExecutorService);

    Long nextWindowStartTimestamp = rateLimiter.getNextWindowStartTimestamp();
    System.out.println("下一个窗口开始时间:" + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(
        new Date(nextWindowStartTimestamp)));

    //token 资源创建预热
    TimeUnit.MILLISECONDS.sleep(200);
    for (int i = 0; i < 10; i++) {
      boolean canShopping = rateLimiter.tryAcquire();
      if (canShopping) {
        System.out.println("购买成功");
      } else {
        System.out.println("购买失败");
      }
    }

    TimeUnit.SECONDS.sleep(2);
    assertFalse(rateLimiter.tryAcquire());


  }

}
```

- 在一个API服务中，为了保护后端服务不被过多请求拖垮，限制每个客户端每秒钟只能发起一定数量的请求
```java
package com.linuxea.impl;

import com.linuxea.RateLimiter;
import java.text.SimpleDateFormat;
import java.util.Date;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;

public class ApiRateLimiter {

  @Test
  public void test() throws InterruptedException {
    ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();
    int maxApiRequest = 50;
    int windowSize = 10;
    //每10秒钟最多允许50个请求, 每秒钟最多允许5个请求
    RateLimiter rateLimiter = new FixedIntervalRateLimiter(maxApiRequest, windowSize,
        TimeUnit.SECONDS,
        scheduledExecutorService);

    Long nextWindowStartTimestamp = rateLimiter.getNextWindowStartTimestamp();
    System.out.println("下一个窗口开始时间:" + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(
        new Date(nextWindowStartTimestamp)));

    //token 资源创建预热
    TimeUnit.MILLISECONDS.sleep(1000);

    Integer successCount = 0;
    for (int j = 0; j < 10; j++) {
      for (int i = 0; i < 10; i++) {
        boolean canShopping = rateLimiter.tryAcquire();
        if (canShopping) {
          successCount++;
          System.out.println("请求成功");
        } else {
          System.out.println("请求失败");
        }
      }
      TimeUnit.SECONDS.sleep(1);
    }

    System.out.println("请求成功次数:" + successCount);

  }

}
```

- 在一个网络游戏中，为了防止玩家利用自动化工具刷经验，限制每个玩家一个自然日能签到一次
```java
package com.linuxea.impl;

import com.linuxea.RateLimiter;
import java.text.SimpleDateFormat;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.ZoneOffset;
import java.util.Date;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;
import org.junit.jupiter.api.Test;

public class NaturalSignRateLimiterTest {

  @Test
  public void test() throws InterruptedException {
    ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor();

    LocalDate today = LocalDate.now();
    LocalDateTime todayStart = LocalDateTime.of(today, LocalTime.MIN);
    // 获取今天零点的时间戳
    long zero = todayStart.toEpochSecond(ZoneOffset.of("+8")) * 1000;

    System.out.println(
        "上一次窗口开始时间" + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(
            new Date(zero)));

    RateLimiter rateLimiter = new FixWindowRateLimiter(1, 1, zero,
        TimeUnit.DAYS, scheduledExecutorService);

    Long nextWindowStartTimestamp = rateLimiter.getNextWindowStartTimestamp();
    System.out.println(
        "下一次窗口开始时间" + new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(
            new Date(nextWindowStartTimestamp)));

    TimeUnit.SECONDS.sleep(1);

    for (int i = 0; i < 10; i++) {
      System.out.println("能否签到" + rateLimiter.tryAcquire());
    }

    scheduledExecutorService.shutdown();
  }

}
```


## 总结

以上三种限流方案各具特点，在实际应用中，需要根据系统的需求、特点和场景来选择合适的限流策略。
- 固定间隔增加令牌适用于需要平滑控制请求速率的场景，但对突发流量处理能力较弱；
- 固定窗口限流适用于突发处理能力要求较高的场景，但存在窗口边界问题；
- 滑动时间窗口限流在平滑流量控制和弹性处理能力方面具有优势，但实现复杂度和计算开销较高。

在选择限流方案时，需要根据具体情况权衡这些因素。


## Reference

- [1] aaaa (https://www.baidu.com)