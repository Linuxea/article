
## linux 上的 tcpdump 命令

tcpdump 是一个可以用来捕获与检查进出系统网络流量的实用工具。
它是网络管理员用来排查问题与安全测试最常用的工具之一。


尽管它的名称叫做 tcpdump， 但你也可以使用它来捕获非 tcp 的流量，比如 UDP， ARP 或者 ICMP。
捕获的包可以写入文件中也可以输入到标准输出流。
tcpdump 最强大的功能之一是它能够使用过滤器捕获你想要分析的数据。

在本文中，我们将介绍如何在 Linux 中使用 tcpdump 命令的基础知识。


## 安装 tcpdump

tcpdump 被大多数 linux 发行版与 macos 默认安装，检查一下你系统上是否安装：

```bash
tcpdump --version
```

输出的格式应该类似如下：

```bash
tcpdump version 4.9.3
libpcap version 1.8.1
OpenSSL 1.1.1d  10 Sep 2019
```

如果 tcpdump 命令不存在你的系统上，tcpdump 命令将出打印输出：
```bash
zsh: command not found: tcpdump
```

这样你可以在你的系统上使用包管理工具进行安装。


### 在 ubuntu 上安装 tcpdump

```bash
sudo apt install tcpdump
```

其他平台请使用相关的包管理工具命令，这里不再详细描述。




## 使用 tcpdump 捕获包

tcudump 的一般语法：
```bash
tcpdump [options] [expression]
```
- 命令选项能够让你控制命令的行为
- 过滤表达式 expression 定义哪些包需要被捕获


只有 root 或者 sudo 优先级的用户可以运行 tcpdump。如果你试图在一个普通用户上使用 tcpdump，你会得到错误信息：
```bash
tcpdump: enp4s0: You don't have permission to capture on that device
(socket: Operation not permitted)
```

tcpdump 最简单的用例是不加选项与过滤来使用：
```
bash tcpdump
```

output
```bash
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp4s0, link-type EN10MB (Ethernet), capture size 262144 bytes
18:23:18.394950 ARP, Request who-has 192.168.158.246 tell 192.168.2.213, length 46
18:23:18.394952 ARP, Request who-has 192.168.158.247 tell 192.168.2.213, length 46
18:23:18.394952 ARP, Request who-has 192.168.158.249 tell 192.168.2.213, length 46

(省略)

44 packets captured
66 packets received by filter
22 packets dropped by kernel
```

tcpdump 会持续捕获包并且把数据输出到标准输出流中，直到它收到中断信号。
使用 <code>Ctrl+C</code> 组合键可以给命令 tcpdump 发送中止信号。



为了更到更详尽的输出，可以传递选项 <code>-v</code>,或者 <code>-vv</code>
```bash
sudo tcpdump -vv
```

你可以指定选项 <code>-c</code>  来指定想要捕获的包的数量。比如，为了捕获且只捕获10个包，你可以这么输入：
```bash
sudo tcpdump -c 10
```

tcpdump 将在达到指包数量时退出。



当没有指定网络接口时，tcpdump 会使用它第一个找到的网络接口并且 dumps 出通过这个网络接口的包。

使用 -D 选项列出 tcpdump 可以从中收集包的可用网络接口：
```bash
sudo tcpdump -D
```

对于每个网络接口，上面的命令会打印出网络接口名称，一个短描述和一个关联数据索引下标：

```bash
➜  ~ sudo tcpdump -D
1.enp4s0 [Up, Running]
2.docker0 [Up, Running]
3.veth1a57e5c [Up, Running]
4.vethe85ca15 [Up, Running]
5.vetha539c20 [Up, Running]
6.veth1f4b317 [Up, Running]
7.veth0164137 [Up, Running]
8.veth1108815 [Up, Running]
9.any (Pseudo-device that captures on all interfaces) [Up, Running]
10.lo [Up, Running, Loopback]
11.br-414a797afd6c [Up]
12.nflog (Linux netfilter log (NFLOG) interface)
13.nfqueue (Linux netfilter queue (NFQUEUE) interface)
14.usbmon1 (USB bus number 1)
15.usbmon2 (USB bus number 2)
```

输出信息展示了当没有指定网络接口时， enp4s0 是 tcpdump 第一个找到并使用的网络接口。
any 网络接口是一个特殊的设备，使用这个接口能够让你捕获所有活动的网络接口。


为了指定你想要捕获的网络接口，调用 tcpdump 命令时使用选项 -i 并在后面跟随特定的网络接口或者关联索引下标。
比如，为了捕获来自网络接口的所有包，你可以使用 any 网络接口：
```bash
sudo tcpdump -i any
```

默认地，tcpdump 会对 ip 地址进行反向 DNS 解析并将端口翻译成名称。使用 -n 选项可以禁止这种翻译：
```bash
sudo tcpdump -n
```

跳过 DNS 查找可以避免产生 DNS 流量并且使用输出更加具有可读性。在使用 tcpdump 时建议使用这个选项。

为了将输出重定向而不是在屏幕上展示，你可以使用重定向选项 > 和 >>:
```bash
sudo tcpdump -n -l | tee file.out
```

上面命令中的 -l 选项告诉 tcpdump 对输出行进行缓冲。 如果不使用此选项，则在生成新行时不会将输出写入屏幕。


## 看懂 tcpdump 的输出

tcpdump 为捕获的每个包起一个新行，根据不同的协议，每行内容包括了时间与包的信息。

典型的 tcp 协议格式行如下：
```bash
[Timestamp] [Protocol] [Src IP].[Src Port] > [Dst IP].[Dst Port]: [Flags], [Seq], [Ack], [Win Size], [Options], [Data Length]
```

逐个字段进行解释如下：
```bash
15:47:24.248737 IP 192.168.1.185.22 > 192.168.1.150.37445: Flags [P.], seq 201747193:201747301, ack 1226568763, win 402, options [nop,nop,TS val 1051794587 ecr 2679218230], length 108
```


- 15:47:24.248737 - 捕获包的本地时间，并且以格式 时：分：秒.frac 展示
- IP - 包的协议。在展示用例中， IP 意味着 Internet protocol version 4 (IPV4)
- 192.168.1.185.22 - 源 ip 地址和端口号，使用 . 进行分隔
- 192.168.1.150.37445 - 目标 ip 地址和端口号，使用 . 进行分隔
- Flags [P.] - TCP 的标志位，在这个例子中，[P.] 意味着 Push Acknowledgment packet, 用来应答上一个包以及发送数据。
其他典型的 tcp 标志位的值如下：
 - [.] - ACK (Acknowledgment)
 - [S] - SYN (Start Connection)
 - [P] - PSH (Push Data)
 - [F] - FIN (Finish Connection)
 - [R] = RST (Reset Connection)
 - [S.] - SYN-ACK (SynAcK Packet)

- seq 201747193:201747301 - 序列号在 first:last 符号中。 它显示数据包中包含的数据数量。 除了数据流中的第一个数据包这些数字是绝对的外，所有后续数据包都用作相对字节位置。 在这个例子中，数字是201747193:201747301，意味着这个数据包包含数据流的字节201747193到201747301。 使用 -S 选项打印绝对序列号。

- ack 1226568763 表示连接的另外一端预期的应答编号

- win 402 - 接收窗口可用的大小，用字节来表示

- options [nop, nop, TS val1051794587 ecr 2679218230] - TCP 的选项值，nop, 或者 ‘no operation’ 是用对填充对齐 TCP 头部大小为 4 字节的倍数。
TS val 是 TCP 时间域，而 ecr 表示一个 echo 回应。 

- length 108 - 载体数据的大小



## tcpdump 过滤器

当没有指定过滤器时，tcpdump 会捕获所有的流量并且产品大量的输出，这让使用者难以分析自己感兴趣的数据。

而过滤器正是 tcpdump 中最强大的功能我之一。它们允许你仅捕获匹配表达式的包。比如在排查 web 服务相关问题时，你可能会考虑使用过滤来只捕获 http 相关的流量。


tcpdump 使用的是 Berkeley Pakcet Filter(BPF) 语法，使用各种加工参数如协议，源与目的ip和端口等来过滤捕获的数据包。


在这篇文章中，我们会看一下最常用的几种过滤器。关于全部的过滤器列表，你可以查看更多相关信息。


### 协议过滤

为了严格捕获特殊协议，可以指定协议为过滤条件。比如，如果你只想要捕获 UDP 流量，你可以这样运行：
```bash
sudo tcpdump -n udp
```

另外一种方式是使用 proto 修饰，后面跟随着协议的数字。
这个命令将会过滤协议数字为 17 的数据包，而且会产生跟上面相同的结果：
```bash
sudo tcpdump -n proto 17
```

关于这个数字 17 的更多信息，可以查看 IP 协议数字列表。


### 主机过滤

为了捕获特定主机关联的数据包，可以使用 host 别名：
```bash
sudo tcpdump -n host 192.168.1.185
```

主机值可以是 ip 地址也可以是名称



### 端口过滤

为了限制包进或者出特定的端口，可以使用 port 修饰。如下命令捕获与端口 SSH（port 22）服务相关的数据包：
```bash
sudo tcpdump -n port 22
```

portrange 修饰符可以捕获一个端口范围的数据包：
```bash
sudo tcpdump -n portrange 110-150
```

### 过滤源与目标

可以基于源与目标主机或者端口来过滤数据包，相关的修饰符为 src， dst。

下面这个命令过滤捕获的是来自主机ip 为 192.168.1.185 的数据包：
```bash
sudo tcpdump -n src host 192.168.1.185
```

为了找到来自任何来源但是目标端口为 80 的数据包，可以使用：
```bash
sudo tcpdump -n dst port 80
```


### 复杂的过滤器

过滤器可以通过运算符号进行组合使用，and（&&）， or（||），和非（！）。

比如，为了捕获来自 ip 地址为 192.168.1.185 的所有 http 流量，可以使用表达式：
```bash
sudo tcpdump -n src 192.168.1.185 and tcp port 80
```


还可以使用括号与分组来创建更加复杂的表达式：
```bash
sudo tcpdump -n 'host 192.168.1.185 and (tcp port 80 or tcp port 443)'
```

为了避免在使用特殊符号时出现解析异常，使用的是单引号括起来。


这里又一个例子是捕获除 ssh 且来自 ip 地址为 192.168.1.185 的数据包：
```bash
sudo tcpdump -n src 192.168.1.185 and not dst port 22
```



## 数据包查看

tcpdump 默认只捕获了头部信息，然而有某些时候可能需要检查数据包的内容。
tcpdump 允许你以 ASCII 和十六进制格式打印数据包的内容

-A 选项告诉 tcpdump 以 ASCII 打印每个包的内容，-X 以十六进制格式：
```bash
sudo tcpdump -n -A
```

使用 -X 选项可打印数据包内容的十六进制与 ASCII 格式：
```bash
sudo tcpdump -n -X
```


## 读写捕获文件

tcpdump 另外一个有用的特性是将捕获数据包保存到文件中。
在捕获数量较大的数据包以便后续分析时，这很方便。

使用 -w 选项跟随文件名可以开始写文件：
```bash
sudo tcpdump -n -w data.pcap
```

上面的命令会将捕获结果保存在以 data.pcap 命名的文件中。你可以按自己意愿命名，不过以 .pcap 拓展名是一个惯例表示捕获包文件。

当使用了 -w 选项时，输出结果将不会在屏幕上展示。
且tcpdump 写入的原生数据包和二进制文件无法使用常规的文件编辑器读取。

为了查看数据文件的内容，在调用 tcpdump 命令时带着选项 -r：
```bash
sudo tcpdump -r data.pcap
```
如果你想要后台运行 tcpdump 命令，可以追加 & 符号在命令末尾。

捕获数据包文件同样可以被其他的一些数据包分析工具打开，比如常用的 Wireshark。

当开启捕获数据包一段较长的时间时，可以开启数据库捕获文件的滚动策略。
tcpdump 允许使用者以一定的周期或者固定的文件大小来创建一个新的数据包文件进行滚动保存。
如下命令，在覆盖旧文件之前，以下命令将创建最多 10 个 200MB 的文件，命名为 file.pcap0、file.pcap1 等。
```bash
sudo tcpdump -n -W 10 -C 200 -w /tmp/file.pcap
```

一旦达到10个文件上限，旧的文件将会被覆盖。

请注意你应该在问题排查期间使用 tcpdump。
如果你想要在指定的时间内运行 tcpdump， 你可以使用 cronjob。 
tcpdump 并没有一个用来在给定时间退出的选项，不过你可以使用 timeout 命令在某些时间退出 tcpdump 命令。
比如，在5分钟后退出，你可以这样写：
```bash
sudo timeout 300 tcpdump -n -w data.pcap
```

## 总结

tcpdump 是一个用来分析与排查网络问题的解决方案。

本文介绍了 tcpdump 的基础语法与使用，如果需要更深入的文档，请查看 tcpdump 网站。


















