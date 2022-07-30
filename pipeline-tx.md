# pipelining and transaction


在不使用 `redis scripting` 时, 如何一定程度上实现脚本同样的原子性与性能?

今天重新简单介绍下 `pipelining` 与 `transaction`, 以及两者在 `redis` 中的应用与组合


## pipelining

> How to optimize round-trip times by batching Redis commands


`Redis` 是基于 `client-server(CS)` 模型的 `TCP` 请求/响应服务

这意味着一个完整的请求响应步骤如下:
- client 发送请求到 server, 读取 socket 以等待 server 的响应
- server 对请求进行处理,然后发送结果回 client 

举个粟子, 四个命令的处理顺序如下:
```zsh
127.0.0.1:6379> incr x
(integer) 1
127.0.0.1:6379> incr x
(integer) 2
127.0.0.1:6379> incr x
(integer) 3
127.0.0.1:6379> incr x
(integer) 4
```

为了减少 `RTT` 以及 `server` 处理请求时在内核态用户态的切换, `pipelining` 管理技术应运而生.

请求/响应模式下, `server` 端可以继续处理新请求,即使 `client` 端还没有读取上一次的响应.

如此这般就可以让 `client` 发送批量请求给 `server` 而不需要等待每个请求的响应, 而是在最后一步中读取所有响应的结果.

这就叫做 `Pipelining`, redis 在早期已经对它实现了支持.

所以不管你正在运行的版本是多少,你都可以使用这项技术. 

接下来我们使用原始的 `netcat` 工具来实践:
```zsh
➜  ~ docker run --rm --name some-redis -d -p6379:6379 redis
f2672c5fe852ddbe2e9868aa474fd5d9ed26f1bf52074d73075056ef544c2491
➜  ~ (printf "incr x\r\nincr x\r\n incr x\r\nincr x\r\n";) | nc localhost 6379
:1
:2
:3
:4

```


### Transactions

> How transactions work in Redis

`Redis` 事务允许执行一组命令, 它的事务提供两个重要保证:
- 事务中的命令按顺序序列化与执行,其他的请求在事务处理过程中不会被响应
- 在客户端执行 `exec` 命令前,事务中所有的命令将不会被执行,一旦调用 `exec` 后才会被全部执行.

举个粟子:
```zsh
➜  ~ docker run --rm --net host -it redis redis-cli -p 6379 
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> INCR foo
QUEUED
127.0.0.1:6379(TX)> INCR foo
QUEUED
127.0.0.1:6379(TX)> exec
1) (integer) 1
2) (integer) 2

```


上述会话很清楚的展示, `exec` 返回了一组响应, 同个事务中的响应与请求发起的顺序是一致的.


### pipelining with Transactions

```
redisCli.Incr("foo")
redisCli.Expire("foo", time.Hour)
```

很多场景下, 在业务代码中如何保证完整执行`incr`, `expire` 两个命令?


这个时候就可以结合 `pipelining` 与 `transaction` 使用
以下用 golang 举例:
```golang
redisCli.TxPipelined(func(p redis.Pipeliner) error {
		ic = p.Incr(key)
		p.ExpireAt(key, time.Hour)
		return nil
	})
```

上面例子相当于一个命令:
```zsh
➜  ~ (printf "MULTI\r\nincr foo\r\nexpire foo 3600\r\nEXEC\r\n") | nc localhost 6379
+OK
+QUEUED
+QUEUED
*2
:7
:1

```


这样在简单的命令场景下,就可以使用到 `pipelining` 技术带来的网络优化以及 `transaction` 所保证的原子性.

除此之外存在一些其他问题, `pipelining` 需要服务端将响应全部暂存内存,如果客户端批量请求命令过多,服务端同时需要占用一定的内存,需要考虑内存的消耗.

## 参考

- [1] [Redis pipelining](https://redis.io/docs/manual/pipelining/)
- [2] [Transactions](https://redis.io/docs/manual/transactions/)
- [3] [https://rafaeleyng.github.io/redis-pipelining-transactions-and-lua-scripts](https://rafaeleyng.github.io/redis-pipelining-transactions-and-lua-scripts)
- [4] [https://stackoverflow.com/questions/29327544/pipelining-vs-transaction-in-redis](https://stackoverflow.com/questions/29327544/pipelining-vs-transaction-in-redis)