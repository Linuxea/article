
## How to Find and Kill Zombie Process in Linux

> 这是一篇关于 linux 上找到僵尸进程然后杀死它们的快速笔记



在谈论关于僵尸进程之前，我们先回忆下 linux 的进程是什么


简而言之，进程是程序的运行实例

- 它可以是以交互进程的形式在前台运行

- 也可以是以后台的形式运行

- 它可以是运行时创建其他进程的父进程

- 也可以是被其他进程创建的子进程


在 linux 上，除了系统进程编号为 0 的第一个初始化 `init` 进程（或者说 `systemd`）
每个进程都有其父进程，进程也拥有属于自己的子进程。




在终端上使用 `pstree` 命令可以查看你系统上的进程树：
```
➜  ~ pstree
systemd─┬─ModemManager───2*[{ModemManager}]
        ├─NetworkManager─┬─dhclient
        │                └─2*[{NetworkManager}]
        ├─accounts-daemon───2*[{accounts-daemon}]
        ├─alsactl
        ├─backlight_helpe───4*[{backlight_helpe}]
        ├─bluetoothd.sh───bluetoothd
        ├─chrome_crashpad───2*[{chrome_crashpad}]
        ├─chrome_crashpad───{chrome_crashpad}
        ├─containerd───14*[{containerd}]


... ...
```



### What is a Zombie process in Linux?

当子进程死亡时，父进程会收到通知以便做一些类似释放内存的清洁工作。
不过父进程如果没有意识到子进程的死亡，子进程会进入僵尸进程的状态。

对于父进程来说，子进程依然存在但是事实上已经死亡。

这就是僵尸进程的诞生由来。




### Do you really need to worry about Zombie processes?

需要重点提一下，僵尸进程并非如他名字听起来一样的充满危险。

问题可能会出现在系统捉襟见肘的内存容量或者太多的僵尸进程吃光了内存。

此外，大多数 linux 进程可以设置最大的进程 id 到 32768.

如果没有可用的进程 id 提供给生产性任务，你的系统可能会崩溃。


这几乎很少发生不过也是存在可能，特别是编码不佳的程序诱导出大量的僵尸进程。


在这种情况下，如果查到并杀死僵尸进程是个不错的主意。




### How to find zombie processes?

linux 上一个进程会处于以下进程状态：

- D = uninterruptible sleep

- I = idle

- R = running

- S = sleeping

- T = stopped by job control signal

- t = stopped by debugger during trace

- Z = zombie



在 linux 上使用 `top` 命令来查看进程以及它们各自的状态：

```
top - 02:29:19 up 2 days,  1:19,  1 user,  load average: 0.34, 0.62, 0.79
Tasks: 293 total,   1 running, 291 sleeping,   0 stopped,   1 zombie
%Cpu(s):  1.0 us,  0.6 sy,  0.0 ni, 98.1 id,  0.2 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :  15894.9 total,   1386.5 free,   3196.9 used,  11311.6 buff/cache
MiB Swap:  16384.0 total,  16384.0 free,      0.0 used.  11960.4 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                    
 2765 linuxea   20   0 3086644  68328  38848 S   3.0   0.4   8:13.10 dde-session-dae                                                                            
 2372 root      20   0 1255476 112532  69212 S   2.7   0.7  15:06.98 Xorg                                                                                       
 2771 linuxea   20   0 3734056 171144 120732 S   2.3   1.1  16:43.78 kwin_x11                                                                                   
19066 linuxea   20   0 2322752 139520  92224 S   2.0   0.9   1:40.56 dde-dock                                                                                   
32231 linuxea   20   0 1311928 105364  85756 S   1.7   0.6   0:26.32 deepin-terminal                                                                            
 2377 root      20   0 1979632  23640  17668 S   1.3   0.1   1:38.33 dde-system-daem                                                                            
24241 linuxea   20   0 1555932 142188  76644 S   1.3   0.9   1:41.97 dde-launcher                                                                               
 2682 linuxea   20   0    8284   5548   3520 S   1.0   0.0   1:48.74 dbus-daemon 



... ...
```


正如上面看到的结果，总共有 293 个进程， 其中 1 个正在运行，291 个正在睡眠，以及 1 个僵尸进程。





现在新的问题出来了，如果杀死僵尸进程？




### How to find and kill a zombie process? Can a zombie process be killed?

僵尸进程早已死亡。如何杀死一个早已死亡的进程？

在僵尸电影中你可以射杀僵尸的头部或者用火烧。不过这里可不提供这种选项。
你可以烧毁你的系统来杀死僵尸进程，不过这并不是一个可行的方案。：）




有些人建议向父进程发送 `SIGCHLD` 信号。 但它更有可能被忽视。
杀死僵尸进程的另一个选择是杀死其父进程。 这听起来很残酷，但这是杀死僵尸进程的唯一可靠方法。




所以，我们首页列出僵尸进程的 id，这通过在终端使用 `ps` 命令可以实现

```bash
➜  ~ ps ux | awk '{if($8=="Z+") print}'
linuxea   5720  0.0  0.0      0     0 pts/2    Z+   02:35   0:00 [zombie] <defunct>
```

`ps ux` 命令输出结果中的第 8 列展示了进程的状态。
你的命令要求打印出所有匹配僵尸状态的进程行。


一旦我们获得僵尸进程的 id。我们可以获得它的父进程 id。
```bash
ps -o ppid = -p <child_id>
```

或者你可以将上面两个命令结合起来，直接打印僵尸进程的 id 以及其父进程 id
```bash
ps -A -ostat,pid,ppid | grep -e '[zZ]'
```


最终，我们得到了父进程的进程 id，通过 `kill` 命令以及上面获取的 id 来杀死进程
```bash
kill -9 <parent_process_ID>
```




最后，你可以使用 `ps` 或者 `top` 命令来验证下是否成功杀死了僵尸进程。



## 参考

- [1] [How to Find and Kill Zombie Process in Linux](https://itsfoss.com/kill-zombie-process-linux/)














































