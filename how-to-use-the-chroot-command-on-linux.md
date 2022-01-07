# How to Use the chroot Command on Linux


使用 `chroot` ，可以在避免跟常规文件系统交互的前提下，设置并运行你的程序或者交互式 `shell`（sh, bash, zsh ...）.

`chroot` 环境中在没有升级到 `root` 权限时不能看到自己特殊的根目录。

这种环境被命名为 `chroot jail`.




## Creating a chroot Environment


我们需要一个目录来扮演 `chroot` 环境中的根目录的角色。

这里创建一个变量用来存储该目录地址，这样我们可以以一种简短的方式来表示对其引用。

至于该目录目前不存在并没有关系，我们会在后面进行创建。如果目录确实存在也行，但是需要保证是一个全新的空目录。


```bash
chr=/home/linuxea/testroot
```

如果目录不存在，我们需要从现在开始创建。

我们可以使用这个命令： `-p` 选项是用来确保同时创建缺失的父目录

```bash
mkdir -p $chr
```


我们需要创建目录来保存我们的 `chroot` 环境对操作系统所需要的部分。

我们将配置并启动一个可以使用 `bash`， 同时包含 `touch`, `rm`, `ls` 命令的最小化 `linux` 环境。

以使我们可以使用 bash 所有内置的命令以及额外的 `touch`， `rm`， `ls` 命令来创建，移除，列表文件。



现在我们切换目录到我们新的根目录。

```bash
cd $chr
```

我们将在最小 linux 环境下需要的二进制文件从常规文件系统的 `/bin` 目录复制到我们的 chroot 的 `/bin` 目录。

- `-v` 选项可以将 `cp` 命令执行明细打印出来
- `--parents` 选项标记在目录下使用完整的文件名


```bash
cp -v --parents /bin/{bash,touch,ls,rm} $chr
```

复制的二进制文件：
```
/bin -> /home/linuxea/testroot/bin
'/bin/bash' -> '/home/linuxea/testroot/bin/bash'
'/bin/touch' -> '/home/linuxea/testroot/bin/touch'
'/bin/ls' -> '/home/linuxea/testroot/bin/ls'
'/bin/rm' -> '/home/linuxea/testroot/bin/rm'
```


不过，这些二进制文本拥有自己的依赖项。

我们需要找出依赖并将它们同时复制到我们的最小化 linux 环境中，否则 `bash`, `touch`, `rm`, 与 `ls` 命令都将不能工作。

我们按顺序对以上选择出来的命令进行操作。

先从 `bash` 开始吧

使用 `file` 命令进行检查：
```bash
file /bin/bash
```
输出结果如下：
```
/bin/bash: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, for GNU/Linux 3.2.0, BuildID[sha1]=1e4b04c843e5befd5ad847201530a5d3d9fc5fba, stripped
```

可以看到 `bash` 是一个动态链接程序。

`ldd` 会为我们列出二进制文件的依赖。

```bash
ldd /bin/bash
```

依赖项被识别并在窗口终端上展示出来：

```bash
ldd /bin/bash
linux-vdso.so.1 (0x00007fffe3498000)
libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6 (0x00007fc85a449000)
libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fc85a444000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fc85a283000)
/lib64/ld-linux-x86-64.so.2 (0x00007fc85a49b000)
```


我们需要将这些文件拷贝到我们的最新环境。

从输出列表中选择出详情信息并一个个复制是耗时且易出错的。

我们可以将其半自动化。我们重新列出相关依赖项并形成一个列表，通过循环来进行复制操作。


我们使用 `ldd` 列出依赖项并将结果通过管道作为 `egrep` 的输入。

- `egrep` 命令的使用跟 `grep -E` 是相同的。

- `-o` 选项用来约束输出来自行中匹配的部分。

- 我们在寻找的是以数字结尾的二进制文件。

```bash
list="$(ldd /bin/bash | egrep -o '/lib.*\.[0-9]')"
```


我们可以通过 `echo` 来检查 `list` 的内容：
```bash
echo $list
```

现在我们拥有了文件列表，可以通过如下循环来逐步遍历，一次复制一个文件。

使用变量 `i` 来作为迭代列表的变量。

```bash
for i in $list; do cp -v --parent $i $chr; done
```


以下是输出结果：
```bash
/lib -> /home/linuxea/testroot/lib
/lib/x86_64-linux-gnu -> /home/linuxea/testroot/lib/x86_64-linux-gnu
'/lib/x86_64-linux-gnu/libtinfo.so.6' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libtinfo.so.6'
'/lib/x86_64-linux-gnu/libdl.so.2' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libdl.so.2'
'/lib/x86_64-linux-gnu/libc.so.6' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libc.so.6'
/lib64 -> /home/linuxea/testroot/lib64
'/lib64/ld-linux-x86-64.so.2' -> '/home/linuxea/testroot/lib64/ld-linux-x86-64.so.2'
```


我们使用这种方式来捕获依赖项，通过循环来进行文件的复制操作。而且只需要少量的编辑就能够完成获取所有依赖的任务。


这里我们使用键盘上面的箭头找回命令，并将编辑为 `touch` 而不是 `bash`。


```bash
list="$(ldd /bin/touch | egrep -o '/lib.*\.[0-9]')"
```

现在重复执行与之前相同的循环操作：

```bash
for i in $list; do cp -v --parent $i $chr; done
```

以下是执行结果：
```bash
'/lib/x86_64-linux-gnu/libc.so.6' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libc.so.6'
'/lib64/ld-linux-x86-64.so.2' -> '/home/linuxea/testroot/lib64/ld-linux-x86-64.so.2'
```


`ls`, `rm` 的操作同样如此。

```bash
linuxea@linuxea-PC:~/testroot$ list="$(ldd /bin/ls | egrep -o '/lib.*\.[0-9]')"
linuxea@linuxea-PC:~/testroot$ for i in $list; do cp -v --parent $i $chr; done
'/lib/x86_64-linux-gnu/libselinux.so.1' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libselinux.so.1'
'/lib/x86_64-linux-gnu/libc.so.6' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libc.so.6'
'/lib/x86_64-linux-gnu/libpcre.so.3' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libpcre.so.3'
'/lib/x86_64-linux-gnu/libdl.so.2' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libdl.so.2'
'/lib64/ld-linux-x86-64.so.2' -> '/home/linuxea/testroot/lib64/ld-linux-x86-64.so.2'
'/lib/x86_64-linux-gnu/libpthread.so.0' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libpthread.so.0'


linuxea@linuxea-PC:~/testroot$ list="$(ldd /bin/rm | egrep -o '/lib.*\.[0-9]')"
linuxea@linuxea-PC:~/testroot$ for i in $list; do cp -v --parent $i $chr; done
'/lib/x86_64-linux-gnu/libc.so.6' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libc.so.6'
'/lib64/ld-linux-x86-64.so.2' -> '/home/linuxea/testroot/lib64/ld-linux-x86-64.so.2'

```


伴随着最后一个命令的依赖项复制到 `chroot` 环境的完成。

我们终于准备好使用 `chroot` 的准备工作。

此命令设置 `chroot` 环境的根目录，并指定作为 shell 运行的应用程序：

```bash
sudo chroot $chr /bin/bash
```

我们的 `chroot` 环境现在处于活动状态。 

你可以注意终端窗口提示也已经发生了更改，同时交互式 shell 由我们环境中的 bash shell 处理。



我们可以使用引入到新环境的命令

```bash
bash-5.0# ls
bin  lib  lib64
```

`ls` 命令在我们的新环境中如预料般工作。

当我们尝试访问环境外的目录时，命令发生失败
```bash
ls /home/linuxea/exists.txt
ls: cannot access '/home/linuxea/exists.txt': No such file or directory
```


结合起来所有命令，我们可以使用 `touch` 来创建文件， `ls` 来列出文件， `rm` 来删除文件。

```bash
bash-5.0# ls
bin  lib  lib64

bash-5.0# touch a

bash-5.0# ls
a  bin  lib  lib64

bash-5.0# rm a

bash-5.0# ls
bin  lib  lib64
```

当然我们还可以使用 `bash` 所提供的内置命令。

比如你在命令行敲下 `help`, `bash` 会有所回应。

```bash
bash-5.0# help
GNU bash, version 5.0.3(1)-release (x86_64-pc-linux-gnu)
These shell commands are defined internally.  Type `help' to see this list.
Type `help name' to find out more about the function `name'.
Use `info bash' to find out more about the shell in general.
Use `man -k' or `info' to find out more about commands not in this list.

A star (*) next to a name means that the command is disabled.

 job_spec [&]                                                                    history [-c] [-d offset] [n] or history -anrw [filename] or history -ps arg >
 (( expression ))                                                                if COMMANDS; then COMMANDS; [ elif COMMANDS; then COMMANDS; ]... [ else COMM>
 . filename [arguments]                                                          jobs [-lnprs] [jobspec ...] or jobs -x command [args]
 :                                                                               kill [-s sigspec | -n signum | -sigspec] pid | jobspec ... or kill -l [sigsp>
 [ arg... ]                                                                      let arg [arg ...]
 [[ expression ]]                                                                local [option] name[=value] ...
 alias [-p] [name[=value] ... ]                                                  logout [n]
 bg [job_spec ...]                                                               mapfile [-d delim] [-n count] [-O origin] [-s count] [-t] [-u fd] [-C callba>
 bind [-lpsvPSVX] [-m keymap] [-f filename] [-q name] [-u name] [-r keyseq] [->  popd [-n] [+N | -N]
 break [n]                                                                       printf [-v var] format [arguments]
 builtin [shell-builtin [arg ...]]                                               pushd [-n] [+N | -N | dir]
 caller [expr]                                                                   pwd [-LP]
 

... ...


```




可以使用 `exit` 来退出 `chroot` 环境：
```bash
bash-5.0# exit
exit
linuxea@linuxea-PC:~/testroot$ 
```


如果你想要移除 `chroot` 环境， 你可以很简单地进行删除：

```bash
rm -r $chr
```



## When Should You Use a chroot?

那么什么时候你应该使用 `chroot`？


`chroot` 环境提供类似于虚拟机的功能，但它是一个更轻量级的解决方案。它也不需要在宿主系统中安装内核，而共享现有的内核。


### Software Development and Product Verification.

开发人员编写软件，产品验证团队（PV）测试软件。

有时候验证团队发现的一些问题无法在开发者电脑上进行复现。

开发者电脑拥有各种工具以及库，这是普通用户与产品验证团队所没拥有的。

通常，适用于开发人员但不适用于其他人的新软件是因为开发人员电脑包含着软件测试版本中所没有的资源。

而 `chroot` 允许开发人员在他们的计算机上拥有一个普通的环境，他们可以在将软件提供给产品验证团队之前将其运行在其中。 可以使用软件所需的最低限度的依赖项来配置运行环境。


### Reducing Development Risk. 


开发人员可以创建一个专用的开发环境，来避免对实际电脑造成混乱与意外。

### Running Deprecated Software. 


有时你不得不运行旧版本的应用。如果旧软件的要求与你的 linux 版本不兼容，可以为其创建一个 `chroot` 环境。


### Ringfencing Applications. 

在 `chroot` 环境中运行 `FTP` 服务器或其他联网设备可以限制外部攻击者能够造成的破坏。 

这可以成为加强系统安全性的重要一步。



## Automate for Convenience


通过这篇文章下来，如果你觉得 `chroot` 对你是有用的，但设置起来有点麻烦，记住你总是能够通过别名，函数，脚本的形式来消除重复性的工作。
而不要因此停止对 `chroot` 的探索。






## 参考

- [1] [How to Use the chroot Command on Linux](https://www.howtogeek.com/441534/how-to-use-the-chroot-command-on-linux/)
- [2] [老司机终于还是翻车了-从零到一手写一个 Docker](https://www.bilibili.com/video/BV1K44y1j7DV)




























