## Background

因为新需求对业务解耦的迫切度更加紧急,上流业务通知下游业务,在业务处理小的情况下使用 RPC 一般能够满足最基础的需求.

但是业务一复杂,整个系统业务的交互会变得更加错综复杂.为了应对业务耦合太严重的问题,首先的想法是使用消息中间件,RocketMq, Kafka等.
但是目前产品技术栈并没有引入相应的中间件,同时为了技术能够更快适合目前的业务量,后面把眼光瞄准了线上正在跑的 Redis.

一开始考虑了 Pub/Sub, 相比 List 能够让多客户端监听到消息.后面又了解到了 Redis Streams, 觉得最佳的方案出来了.
本文即是学习与使用 Redis Streams 的介绍.




## Redis Streams

Streams 是 Redis 5.0 版本引入的数据类型,是以更抽象的方式对日志数据的建模.
其日志本质是依然是完整的:以一种追加写入的方式. Redis Streams 是以只追加为主的数据类型.
同时作为内存存在的抽象数据类型,Redis Streams 实现了更加强大的操作来克服日志文件的限制.

Streams 作为 Redis 中最复杂的数据类型,得益于实现了额外且非强制的特性,一系列阻塞操作和称之为消费组的东东.

消费组最初是由 kafka 引入的概述, Redis 以全新的方式重写达到相似理念.不过放心,大家的目标是相同的,就是让一组客户端能够以不同的视角来合作消费流中的消息.


## Streams Basic

为了专注于 Streams 数据结构本身,先忽略其高级特性.而是先从操作与访问的命令入手.

这跟其他大多数数据类型是相通的,比如列表,集合,有序集合等等.
对了,列表同样有阻塞的api, 所以其实从这个角度来看,Streams 其实并没有太大的区别.

因为 Streams 是只追加的数据类型,其基础的写命令 XADD 后面跟随着称之为条目(下方也称之为条目吧)的东西来追加到指定的 stream.
条目不是字符串,而且由一个或多个键值对组成的具有结构化的数据.

```bash
> xadd user * id 1 name linuxea
"1630566672165-0"
```

如上命令,给指定的 stream user 添加了信息: id 为1, name 为 linuxea 的条目.并且使用了自动增长的id表示.命令返回的唯一结果是条目生成的 id.
当然了我们也可以指定自己的id.但是记住 Streams 是只追加的数据类型, id 增长的特性是不能被破坏.你不能添加比旧条目 id 更小或等于的id.

```bash
> xadd newuser 1 id 1 name linuxea
"1-0"
> xadd newuser 1 id 1 name linuxea
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item
```

加完之后如果想获得指定 stream 的长度,可以这样:
```bash
> xlen newuser
(integer) 1
```

Redis 是内存数据库,stream 在这个基础同样不能无限增长.需要在适当时对 stream 进行裁剪.
```bash
// ... 先 XADD 添加多一点数据
// 然后
> xtrim newuser MAXLEN 5
(integer) 11
```
返回的是裁剪掉的数量.




现在回过头来看看 id 的组成:
<millisecondsTime>-<sequenceNumber>

id 的前半部分是 redis 节点的当前时间,后半部分是用来区分在同一时间生成的消息的增长序号.这样就可以唯一的标识一条条目.

如果节点时间发生了时钟回跳,那也没关系.上一个条目时间会取代当前条目时间,这样可以保证id增长特性依然成立.序列号在这个场景下也应用到.
序列号是64位宽,这意味着实际应用中哪怕同一时间内,消息条目数量是没有上限瓶颈的.




## Getting data from Streams

现在我们可以通过 XADD 向指定 stream 追加条目了.不过从 stream 里面取得条目可能就没那么显而易见了.

下面介绍三种方式来从 stream 获取数据.


### XRANGE & XREVRANGE

顾名思义,是以一个范围来取得 stream 内的条目.而所需要的参数即是条目的id.包括起始,结束id.
```bash
> xrange newuser - + 
1) 1) "1-0"
   2) 1) "id"
      2) "1"
      3) "name"
      4) "linuxea"
```

xrange 后面跟随指定的 stream, - 的含义表示一个最小的id, + 号则表示一个最大的id.
因为我们刚刚只放了一个条目.所以你能看到的结果如上.返回的是一个数组,每个数组元素包含id与真实的条目内容.

试一下指定id来查找:
```bash
> xrange newuser 1-0 + 
1) 1) "1-0"
   2) 1) "id"
      2) "1"
      3) "name"
      4) "linuxea"
```

显而易见,结果是一样的.

如果遇到 stream 条目特别多时,可以加上一个可选参数 COUNT 来约束返回的最大条目.

```bash
> xrange newuser 1-0 + count 10
1) 1) "1-0"
   2) 1) "id"
      2) "1"
      3) "name"
      4) "linuxea"
```

显而易见,结果是一样的.因为我就只有一个.

另外一个命令 XREVRANGE,顾名思义,是以相反的顺序返回条目.可能唯一的使用场景是查询最后一个元素是什么.

```bash
> xrevrange newuser + - count 10
1) 1) "1-0"
   2) 1) "id"
      2) "1"
      3) "name"
      4) "linuxea"
```

显而易见,结果也是一样的.

如此来看, range 查询可能更多的是把 stream 当作一个时光机,根据你想要的时间范围来回溯历史数据,只需要从口袋一掏,就能够查看历史数据.




### Listening for new items with XREAD

不想查询历史范围数据,只想要最新的消息?
用 XREAD.

XREAD 在这个概念上跟 Pub/Sub 和 List 是相似的.
不过本质上还是有不同:
- Pub/Sub 消息是阅后即焚,不被存储的.stream 是有记忆的,从消费者角度来看,它们保有最后消费的条目 id, 这让他们能够知道什么才是"新"消息.
- List 的消息只能到达一个客户端就会被 POP 弹出移除.
- Streams 的消费组提供更高级别的控制:处理消息的应答,检查待办条目,认领未处理条目,客户端一致的历史可见性.

```bash
> xread count 2 streams newuser 0
1) 1) "newuser"
   2) 1) 1) "1-0"
         2) 1) "id"
            2) "1"
            3) "name"
```

XREAD 用来监听到达 stream 的消息.
count 是一个非强制选项,用来约束一次读取的条目数目.
streams 是一个强制选项,用来指定读取的 stream .
0 表示只获取比指定 id 0 大的条目.

所以上面这条命令意思就是获取指定 stream newuser 中比 0-0 大的数量为2的条目列表.

streams 选项表明了其支持多个 stream, 同时需要注意对应好 id 
```bash
> xread streams newuser user 0 0
1) 1) "newuser"
   2) 1) 1) "1-0"
         2) 1) "id"
            2) "1"
            3) "name"
            4) "linuxea"
2) 1) "user"
   2) 1) 1) "1630566672165-0"
         2) 1) "id"
            2) "1"
            3) "name"
            4) "linuxea"
```

因为这个原因,所以 streams 选项只能放在最后了.


如果这么看的话, XREAD 支持多 stream 获取新消息可能跟 xrange 相比没有特殊的地方.不过,有趣的一部分来了,一个 BLOCK 可选选项能够让我们把 XREAD 轻易变成一个阻塞命令.
```bash
> xread block 0 streams newuser $
```

block 选项表示阻塞超时时间(0 表示永远阻塞不超时),同时使用的 id 是特殊的 $ .它表明一个 stream 中存在的最大 id, 这表明这个命令只会接收当下最新的消息.是不是跟日志 tail -f 在某种程度上是相似的.

当使用 BLOCK 选项时,特殊的 ID $ 并不是必须的.我们同样可以使用真实有效的 id, 当满足条件时阻塞会被解除,否则会阻塞到指定超时时间.

XREAD 不加选项 COUNT 和 BLOCK 时,是一个非常基础的命令用来附加消费者到一或多个 stream.



### Consumer Groups

多个客户端手头上的任务是来自同一个 stream 时, XREAD 提供了将条目发放到多个客户端的能力,同时潜在地提供读的扩展性,只需要加客户端即可.
然而,中心问题可能是我们需要的不是这样.我们需要的是为不同的客户端提供不同的消息子集.

一个明显且有用的场景是,stream 消息是重量级且难以处理的.通过多个 worker 可以横向拓展消息处理能力,将消息路由到不同的 worker 来实现完成更多的工作.

在现实场景下,想像我们有三个客户端 C1, C2, C3, stream 包含的条目有 1,2,3,4,5,6,7,采用下图来表明我们想要的处理消息意图:
```bash
1 -> C1
2 -> C2
3 -> C3
4 -> C1
5 -> C2
6 -> C3
7 -> C1
```


Streams 采用了消费组的概念.消费组提供了如下的保证:
- 每个消息只会分发到同一个消费者,不会存在两个消费者共享同一个消息.
- 消费组内每个消费者的区分是名称,而且是大小写敏感.
- 每个消费组拥有一个概念上的未被消费的条目 id, 当一个消费者想要请求新消息时,它会得到一条从未分发的消息.
- 要求一个明确的 ack 来表示一条消息被正确处理,以致此消息可以从消费组中移除.
- 消费组会跟踪所有处于待处理的消息,也就是那些被分发却没有收到应答的消息.幸亏这个特性,当访问 stream 历史消息时,每个消费者只能看到分发给他们的消息.


由此来看,一个消费组可以想像成关于 stream 的状态集合.

+----------------------------------------+
| consumer_group_name: mygroup           |
| consumer_group_stream: somekey         |
| last_delivered_id: 1292309234234-92    |
|                                        |
| consumers:                             |
|    "consumer-1" with pending messages  |
|       1292309234234-4                  |
|       1292309234232-8                  |
|    "consumer-42" with pending messages |
|       ... (and so forth)               |
+----------------------------------------+


现在放大来讲讲具体关于消费组的命令:
- XGROUP 创建,销毁,管理 消费组
- XREADGROUP 通过消费组从 stream 获取消息
- XACK 标记待办列表中消息为正确处理


```bash
> XGROUP CREATE newuser mygroup $
OK
```

- 上面命令创建了一个带有指定 id 的消费组, 例子中的是'$', 这这意味着从创建消费组开始新的消息才会被提供组内的消费者.
- 我们也可以指定 0 来表示将历史的所有数据分发给组内的消费者.
- 当然你也可以指定一个真实有效的 id.

XGROUP CREATE 同样支持在 stream 不存在时自动创建:
```bash
XGROUP CREATE newuser mygroup $ MKSTREAM
OK
``` 


```bash
XREADGROUP GROUP mygroup alice STREAMS newuser >
```

上述命令通过消费组 mygroup 中的消费者 alice 指定消费 stream newuser 的没有分发给其他同组消费者的消息.


id 的不同
- 当指定 > 时,返回未分发给其他消费者的消息.作为后续影响,会更新消费组的 last ID 信息
- 当指定另外的有效 ID, 这个命令会让我们访问历史的待处理列表.



```bash
> XACK newuser mygroup 1-0
(integer)1
```

当通过消费组消费完消息时,需要将对应的消息 id 标志为处理.将此消息从待处理列表中移除.

ACK 的意义在于保证数据被用户明确标记为处理完成.如果对于数据丢失的容忍度高或者重要性低,可以将 ack 特性关闭.
```bash
XREADGROUP GROUP mygroup consumer NOACK STREAMS newuser >
```

查看组内各个消费者待处理列表的信息:
```bash
> XPENDING newuser mygroup
1) (integer) 15
2) "1-0"
3) "16305704629832-7"
4) 1) 1) "consumer1"
      2) "14"
   2) 1) "consumer2"
      2) "1"
```


还有关于查看消费组信息的命令 XINFO 等.不再继续描述,实际应用场景读者也会更加深入的了解.



## 总结
Redis Stream 作为一款较量级的消息中间件,同样具备了成熟的形态.
引进成本较低,Api 也非常简洁.对于业务的异步,解耦,削峰能够处理得很好.
设计思想对于消息队列初学者具备启发意义.



## 参考

- [1] [Introduction to Redis Streams](https://redis.io/topics/streams-intro)

























 
