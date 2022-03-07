In this article, let’s take a look at the performance indicators of the disk and how to observe these indicators.

在本文中，我们来看看磁盘的性能指标以及如何观察这些指标。


## Linux Disk Performance Metrics

When it comes to measuring disk performance, we often mention five common indicators: utilization, saturation, IOPS, throughput, and response time. These five indicators are the basic indicators to measure disk performance.


当谈论到磁盘的性能时，我们经常提到五个普遍的指标：utilization, saturatin, IOPS, throughput 和 response time.
这五个指标是评估磁盘性能的基本。


Utilization: The percentage of time the disk is processing I/O. Excessive usage (such as more than 80%) usually means that there is a performance bottleneck in disk I/O.
Saturation: Refers to how busy the disk is processing I/O. Excessive saturation means that the disk has a serious performance bottleneck. When saturation is 100%, the disk cannot accept new I/O requests.
IOPS (Input/Output Per Second): Refers to the number of I/O requests per second.
Throughput: The size of I/O requests per second.
Response time: Refers to the interval time between sending an I/O request and receiving a response.


- Utilization: 磁盘处理 I/O 的时间百分比，过多的使用率（比如大于 80%）通常意味着存在磁盘 I/O 瓶颈
- Saturation: 指磁盘处理 I/O 的繁忙程序。过度饱合意味着磁盘存在严重的性能瓶颈。当当饱和度为 100% 时，磁盘无法接受新的 I/O 请求。
- IOPS (Input/Ouput Per Second)：指每秒 I/O 请求数量。
- Throughput: 每秒 I/O 请求的大小。
- Response time: 指发出 I/O 请求到接收响应之间的时间间隔。


It should be noted here that utilization only considers the presence or absence of I/O, not the size of the I/O. In other words, when the utilization is 100%, it is still possible for the disk to accept new I/O requests.

 这里要提到的是，利用率只考虑 I/O 的有无，而不是 I/O 大小。也就是说，当利用率达到 100% 时，磁盘依然可能接收新的 I/O 请求。


Generally speaking, when you select a server for an application, you must first perform a benchmark test on the I/O performance of the disk, so that you can accurately assess whether the disk performance can meet the needs of the application.

通常来讲，当你为应用程序选择服务器时，你必须首先对磁盘的 I/O 性能进行基准测试，以便你能够精准评估磁盘性能是否能够满足应用程序的需要。


Of course, this requires you to test the performance of different I/O sizes (usually several values ​​between 512B and 1MB) in various scenarios such as random read, sequential read, random write, and sequential write.


当然，这需要你在随机读、顺序读、随机写、顺序写等各种场景下测试不同I/O大小（通常是512B到1MB之间的几个值）的性能。



## Disk I/O Observation

The first thing to observe is the usage of each disk. iostat is the most commonly used disk I/O performance observation tool. It provides various common performance indicators such as the usage, IOPS, and throughput of each disk. Of course, these indicators actually come from /proc/diskstats.

首要需要观察的就是磁盘的使用情况。 iostat 是最常用的 I/O 磁盘性能观察工具。
它提供了许多常用的性能指标如使用率， IOPS 与吞吐量。当然，这些指标事实上是来自于 /proc/diskstats.

Following is an example output of iostat :

如上是 iostat 的示例输出：

```bash
➜  ~ iostat -d -x 1 
Linux 5.10.83-amd64-desktop (linuxea-PC)        2022年03月07日  _x86_64_        (6 CPU)

Device            r/s     w/s     rkB/s     wkB/s   rrqm/s   wrqm/s  %rrqm  %wrqm r_await w_await aqu-sz rareq-sz wareq-sz  svctm  %util
loop0            3.94    0.00    185.92      0.00     0.00     0.00   0.00   0.00   11.63    0.00   0.05    47.17     0.00   9.35   3.68
sda              0.08    0.00      2.93      0.00     0.00     0.00   0.00   0.00    0.40    0.00   0.00    35.41     0.00   0.67   0.01
sdb             29.50   17.52   1121.62   1584.41    17.02    21.28  36.59  54.84   26.45   25.28   1.33    38.02    90.44   3.91  18.36
```

Among the above indicators, you should pay attention to:

在上面的指标中，你需要关注的是：

%util is the disk I/O usage we mentioned earlier
r/s+ w/s is IOPS
rkB/s+wkB/s is the throughput
r_await+w_await is the response time

- %util 是磁盘 I/O 使用情况 (usage)
- r/s+ w/s 是 IOPS
- rkB/s+wkB/s 是吞吐量 (throughput)
- r_await+w_await 是响应时间 (response time)

You may have noticed that disk saturation is not directly available from iostat. In fact, there is usually no other simple way to measure saturation. However, you can compare what you observe, the average request queue length or the wait time for read and write requests to complete, with the results of benchmark tests (such as through fio) to synthesize Evaluate disk saturation.


你应该注意到磁盘指标 saturation 饱合度 并没有从 iostat 命令中直接得到。
事实上，也不存在其他简单方法来评估饱合度。不过，你可以比较你所观察到的，平均请求队列长度或者读写请求完成的等待时间，用基准测试的结果（比如通过fio）来综合评估磁盘饱和度。

## Process I/O observation

In addition to the I/O situation of each disk, the I/O situation of each process is also the focus of your attention.
The iostat mentioned above only provides the overall I/O performance data of the disk. The disadvantage is that it is impossible to know which processes are reading and writing to the disk. To observe the I/O of a process, you can also use the tools pidstat and iotop.


除了每个磁盘的I/O情况，每个进程的I/O情况也是大家关注的重点。

上面提到的 iostat 仅仅提供了全局的 I/O 性能数据。缺点是无法知道哪个线程正在对磁盘进行读写。为了观察进程的 I/O，你可以使用工具如 pidstat 与 iotop.

For example, to use pidstat
举个粟子，使用 pidstat:

```bash
➜  ~ pidstat -d 1 
Linux 5.10.83-amd64-desktop (linuxea-PC)        2022年03月07日  _x86_64_        (6 CPU)

11时26分59秒   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
11时27分00秒     0       829     -1.00     -1.00     -1.00       3  jbd2/sdb7-8

11时27分00秒   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
11时27分01秒  1000     16547      0.00     24.00      0.00       0  feishu

11时27分01秒   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
11时27分02秒  1000      4521      0.00      4.00      0.00       0  lantern
11时27分02秒  1000      5657      0.00      4.00      0.00       0  lantern

11时27分02秒   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
^C

平均时间:   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s iodelay  Command
平均时间:     0       829     -1.00     -1.00     -1.00       1  jbd2/sdb7-8
平均时间:  1000      4521      0.00      1.00      0.00       0  lantern
平均时间:  1000      5657      0.00      1.00      0.00       0  lantern
平均时间:  1000     16547      0.00      5.97      0.00       0  feishu
```


As you can see from the output of pidstat, it can view the I/O situation of each process in real time, including the following:
User ID (UID) and Process ID (PID).
The size of data read per second (kB_rd/s), in KB.
The size of the write request data issued per second (kB_wr/s), the unit is KB.
The data size of canceled write requests per second (kB_ccwr/s), in KB.
Block I/O delay (iodelay), including the time to wait for the completion of synchronized block I/O and swap-in block I/O, in clock cycles.

正如你所看到的，pidstat 可以实时查看每个进程的 I/O 状况，包括如下信息：
- User ID(UID) 与 Process ID(PID)
- 每秒读数据大小(kB_rd/s)
- 每秒发出的写请求数据大小（kB_wr/s）
- 每秒取消写入请求的数据大小 (kB_ccwr/s)
- 阻塞 I/O 延迟 (iodelay)，包括等待同步 I/O 和换入 I/O 完成的时间，以时钟周期为单位。



In addition to real-time viewing with pidstat, sorting processes according to I/O size is also a common method in performance analysis. For this, I recommend another tool, iotop. It’s a top-like tool in that you can sort processes by I/O size and find those with a larger I/O.



除了使用 pidstat 实时查看之外，根据 I/O 大小对进程进行排序也是性能分析中常用的方法。
因此我推荐另外一个工具 iotop.

这是一个类似 top 的工具，你可以通过 I/O 大小来对进行进行排序，并找到具有更大 I/O 的进程。


xxxx
（放图好了）


From this output, you can see that the first two lines represent the total disk read and write size of the process and the total disk real read and write size, respectively. They may not be equal due to factors such as caches, buffers, I/O coalescing, etc.
The remaining part represents the I/O situation of the process from various perspectives, including thread ID, I/O priority, disk read size per second, disk write size per second, swap in and wait for I/O Clock percentage.



从输出中，你可能看到头两行各自代表：进程磁盘读写总大小以及磁盘实际读写总大小。
可能因为一些因素如 cache, buffer, I/O 合并等，导致两者并不相等。

剩下的部分从各个角度表示进程的I/O情况，包括线程ID、I/O优先级、每秒磁盘读取大小、每秒磁盘写入大小、换入和等待I/O时钟百分比。


Conclusion
In this article, I introduced performance metrics and performance tools for Linux disk I/O. We usually use several indicators such as IOPS, throughput, utilization, saturation and response time to evaluate the I/O performance of the disk.
You can use iostat to get the I/O situation of the disk, and you can also use pidstat, iotop, etc. to observe the I/O situation of the process. However, when analyzing these performance indicators, you should pay attention to the comprehensive analysis combined with the read/write ratio, I/O type, and I/O size.

## Conclusion

在这篇文章中，介绍了 linux 性能指标以及工具.
我们通常使用几个指标如 IOPS, throughput, utilization, saturation 与 response time 来评估磁盘的 I/O 性能。

可以使用 iostat 来获取磁盘的 I/O 状况，为了观察进程的  I/O 状况你也可以使用 pidstat, iotop 等等。

但是在分析这些性能指标时，你需要注意结合读比写，I/O 类型与大小来综合分析。