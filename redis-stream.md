
## Redis Streams 介绍

Streams 是 Redis 5.0 引入的一种新的数据类型, 以一种更加抽象的方式对日志数据进行建模.不过日志的本质依然是完整的,以一种追加的形式打开文件, 
Redis Streams 以仅追加为主的数据结构.

至少在概念上,因为作为的是内存上的抽象数据类型,因为 Redis Streams 实现了更加强大的操作来克服日志文件的限制.

尽管其本身的数据结构十分简单,但是事实上其拥有额外的实现:一系列阻塞操作来使消费者能够等待直到生产将从另一端将数据加入到 stream 中,和称之为消费组的概念等.
这使得 Streams 成为了 Redis 中最复杂的数据类型.

消费组最开始是由非常受欢迎的消息系统 kafka 引入的.Redis 以一种完全不同的条款实现了相似的概念,
但是目标是一致的: 允许一组客户端来合作消费同一消息 stream 的不同部分.




## Streams 基础

为了达能够理解 Redis Streams 以及如何使用的这个目标,我们会先忽略其高级特性, 而是其命令操作与访问, 专注于其数据结构本身.

这是基础的一部分,并且与大多数其他的数据类型,如 Lists, Sets, Sorted Sets 等等是相似相通的.

对了,注意到 Lists 同样拥有额外复杂的阻塞 API,如BLPOP 等类似名称来导入使用.

所以这方面, Strems 与 Lists 并没有太大区别,只是增加的 API 更加复杂与强大.

因为 Streams 是仅追加的数据类型, 基础的写命令,如 XADD ,追加新的数据到指定的 stream.
一个 stream 的条目并不是仅仅字符串,而是由一个或者多个键值对组成的.
这样, stream 每个实体都是结构化的.

```bash
XADD mystream * name linuxea age 19
1518951480106-0
```

上面的 XADD 命令调用是添加一个条目 name:linuxea, age:19 到 stream, 调用结构唯一的返回是一个自动生成的实体 id, 具体值是 1518951480106-0.

XADD 使用了第一个参数作为 stream 命令,第二个参数是用来区分每个实体条目的id.
然而,上面的命令我们使用了 * 作为参数, 因为我们想要 redis 服务端为我们生成一个新的自增长的 id, 因此后面添加的新实体条目将会拥有比旧的实体条目拥有更大的 id.

让服务端自动生成一个 id, 这是你绝大多数情况下想要的, 要明确指定一个 id 的理解不多见.这个我们可以稍后再安利.

事实上 stream 的 id 是另外一个跟日志文件相似的地方, 比如日志文件行号,或者说在日志文件中的偏移量值,这能够用来标识每一个给定的实体条目.

可以得到 stream 中实体条目的数量, 使用的是 XLEN 命令:
```bash
XLEN mystream
(integer) 1
```


实体条目 IDs

XADD 命令返回的实体条目 id 用来明确的标识 stream 中每个实体条目.它由两部分组成:

<毫秒时间>-<序列号>


毫秒时间这部分事实上来自本地 redis 节点的本地时间, 如果当前的节点时间恰好小于上一个实体条目的时间部分,上一个实体条目的时间会被使用.
所以如果时钟向后跳，单调递增的 ID 属性仍然成立。

<序列号>用来在相同毫秒内使用.因为序列号是 62位 宽,所以在相同毫秒内生成的实体条目数量是不会有限制的.


id 的格式组成第一眼看上去可能有的奇怪, 温和的读者可能会好奇为什么时间是id的一部分.
原因是因为 Redis Streams 支持根据id的范围查询. 因为id跟实体生成的时间相关,这同样给予根据时间范围查询的能力而不需要其他付出.
我们将会在谈到 XRANGE 命令时看到这点.

如果因为某些原因用户需要自定义关联外部系统id而不需要跟时间有关系.根据上面提到的, XADD 命令可以带上明确的 id 而不是用通配符 * 来代替.
就像这样:
```bash
XADD somestream 0-1 filed value
0-1
XADD somestream 0-2 filed value
```


在这个案例中,请注意最小的id 是 0-1,并且 XADD 命令不会再接收小于等于上一个id:
```bash
XADD somestream 0-1 foo bar
(error) ERR The ID specified in XADD is equal or smaller than the target stream top item
```
这跟 Streams 作为追加的数据结构是冲突的.



## 从 Redis Streams 中获取数据

通过 XADD 最终我们实现了追加实体到 stream.追加数据到 stream 是直接显然的,从 stream 中查询以提取数据却不是那么明显.
如果我们继续以日志文件做比喻,一个显而易见的方式是模仿我们一般使用的 unix 命令 tail -f, 即我们可以开始监听以获取得最新的添加到 stream 的消息.
注意下这跟列表的阻塞操作不一样,比如 BLPOP 当一个元素达到客户端时会以弹出形式离开列表.使用 Streams 我们想要的是多个客户端能够看到最新追加到 stream 的消息.
(这跟tail -f 多进程下都能看到追加的日志数据一样).

使用传统的术语,我们希望流能够将消息像扇子一样扇出到多个客户端.


然而,这也只是其中一种流的访问模式.
我们同样可以以一种完全不同的方式:不是作为一个消息系统,而只是一个时间序列存储.
在这个场景下,对于获取最新的数据同样有用,不过更自然的模式是通过时间范围来查询.或者以游标的形式来回溯整个历史数据.
这绝对是另一种有用的访问模式。


最后,如果我们从消费者的角度来看待 stream, 我们可以会希望以另外的一种方式访问 stream.
stream 中的消息能够被分区到多个不同的消费者消费,因此由消费者们组成的群组只能看到到达 stream 消息的其中一个子集.
通过这种方式,可以通过跨不同的消费者来拓展消息的处理能力,而不是由一个消费者来处理所有的消息.


Redis Streams 通过不同的命令能够支持上面提到所有三种查询模式.
下一章节将会展示它们.



## 通过范围查找
对于 stream 的范围查找我们只需要指定两个 id 参数:开始与结束.
返回的范围包含如果存在的开始与结束消息,因此这是一个闭区间的.
有两个特别的id - 与 +, 表示可能最小的与最大的id.

```bash
XRANGE mystream - +
1215) 1) "1630425570893-0"
      2) 1) "vipContinueDays"
         2) "47"
         3) "userId"
         4) "46198"
1216) 1) "1630425570904-0"
      2) 1) "userId"
         2) "46220"
         3) "vipContinueDays"
         4) "1"
1217) 1) "1630475812110-0"
      2) 1) "userId"
         2) "1"
```

每个实体条目以一个数组两个元素的形式返回:id 与 键值对列表.
我们说过 id 与时间是关联的,因为 id 的左半部分表示实体创建时的本地节点时间.
这意味着可以通过时间范围来查找.
为了实现这个目的,我们想要忽略掉 id 关于序列号的部分, 如果忽略序列号,范围的开始序列号为认定为0, 结束的序列号为被认定为最大的序列号.
这样的话查询只需要两个毫秒时间,我们可以获取在这段时间内的所有实体以一个闭区间的模式.
举个粟子:
```bash
> XRANGE mystream 1518951480106 1518951480107
1) 1) 1518951480106-0
   2) 1) "sensor-id"
      2) "1234"
      3) "temperature"
      4) "19.8"
``

在这个范围内我只有一个实体,但是实际环境下,真实数据可能很多或者时间范围很大,都会造成结果变得非常巨大.
因此, XRANGE 支持一个最后的可选选项 COUNT, 来取前 N 个元素.
如果你之后想要更多, 可以拿上一次返回的最大 id 做为新的开始参数, 以此来循环查询.

```bash
XRANGE mystream - + COUNT 2
1) 1) 1519073278252-0
   2) 1) "foo"
      2) "value_1"
2) 1) 1519073279157-0
   2) 1) "foo"
      2) "value_2"
```


上面提到过范围是个闭区间,为了方便不出现重复数据,可以定义为一个半开闭区间:
```bash
> XRANGE mystream (1519073279157-0 + COUNT 2
1) 1) 1519073280281-0
   2) 1) "foo"
      2) "value_3"
2) 1) 1519073281432-0
   2) 1) "foo"
      2) "value_4"
```


XRANGE 的查询复杂度为 O(log(N)), 返回元素数量的复杂度为 O(M), 对于一个小数量 COUNT 选项, XRANGE 依然具有 O(log(N)) 的时间复杂度,
这意味着每一步的迭代都是快速的.
因此 XRANGE 是真正意义上的流式迭代并且不需要 XSCAN 命令支持.


XREVRANGE 等同于 XRANGE 但是是以相反顺序返回.因此 XREVRANGE 的真实使用场景一般是检查 stream 的最后一个元素是什么.

```bash
> XREVRANGE mystream + - COUNT 1
1) 1) 1519073287312-0
   2) 1) "foo"
      2) "value_10"
```


注意的是 XREVRANGE 命令将开始与结束的参数位置反转了.





## 使用 XREAD 监听新数据条目

当我们不想以范围的方式来访问 stream, 通常想要的是 stream 的新消息.
这个概念与 Pub/Sub 相关,订阅一个频道或者监听一个阻塞队列,但是消费 stream 本质上还是有不同的地方:

1.一个 stream 可以有多个客户端.对于每个新消息默认会送达到每个等待中的消费者.这跟阻塞队列是不同的,阻塞队列的消费者获取到的是不同的数据.
而这个能力跟发布/订阅是相似的.

2.然而发布/订阅的消息内容是不会在任何情况下存储,阻塞队列当消息被接收后会从列表中弹出删除.
stream 则是以完全不同的基础方式工作,所有消息都无限期地附加在流中, 除非用户明确删除.
不同的消费者从他们各自的角度来记住最后的消息id,因此它们知道哪些对于他们来说是新消息.

3.streams 消费组提供发布/订阅与阻塞列表无法达到的控制水平, 对于相同的stream 下的消费组,处理消息的明确确认,检查待办列表,认领未处理消息,每个客户端的历史一致可见性,











