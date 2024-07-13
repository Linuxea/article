## 背景

最近自建的 rabbitmq 所在的 ec2 实例，open-falcon 经常在高峰期出现 cpuidea 过低的告警。今天周六在家闲得无聊，好奇就看看。

## 排查

### aws ec2 metrics

![cpuutilization](cpuutilization.JPEG "cpuutilization")

如上图，cpu 使用率实际并不高，高峰期也没有上涨特别厉害，整体稳定维持在 50% 上下。


### open-falcon metrics

open-falcon 统计的 cpuidle 存在差异，看着是把 cpu 处于 iowait 状态也计算到非空闲状态 ([falcon 指标说明](https://github.com/open-falcon/book/blob/master/en_0_2/faq/linux-metrics.md))：（如下两图，体现两个指标的反向关联）:

![cpuidle.JPEG](cpuidle.JPEG "cpuidle.JPEG")

![cpuiowait.JPEG](cpuiowait.JPEG "cpuiowait.JPEG")

### aws ec2 storage

查看 rabbitmq 实例挂载的硬盘：

![iowrite.JPEG](iowrite.JPEG "iowrite.JPEG")

只看<strong>写</strong>操作相关的统计，吞吐量在高峰期与平时存在的差异体现出来是合理的。

而<strong>读</strong>操作次数与吞吐量，更多的可能是 rabbitmq 本身的特性，当消息被发送到 RabbitMQ 时，它们首先被写入到磁盘以确保消息的持久性。这一过程导致写操作的数量和吞吐量较高。

而消息读取：消费者从RabbitMQ读取消息时，如果消息已被预先加载到内存或使用缓存机制，那么对磁盘的读取操作可能较少。这体现在平时非高峰期，足够的内存与缓存，读相关的操作统计并不明显。

### summary

所以得出初步结论：原因非常大可能还是 cpu 处于大量的 iowait 状态，触发 open-falcon cpu idle 低告警。而 cpu iowait 值大的原因还是在于磁盘的吞吐性能太差，在高峰期这个问题被放大：
![writeOps.PNG](writeOps.PNG "writeOps.PNG")

![iops.png](iops.png "iops.png")


## 参考
- [Basic collection item of Linux Operation](https://github.com/open-falcon/book/blob/master/en_0_2/faq/linux-metrics.md)
- [Instance metrics](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/viewing_metrics_with_cloudwatch.html#ec2-cloudwatch-metrics)