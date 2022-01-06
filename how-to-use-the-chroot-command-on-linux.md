
# How to Use the chroot Command on Linux


With chroot you can set up and run programs or interactive shells such as Bash in an encapsulated filesystem that is prevented from interacting with your regular filesystem. Everything within the chroot environment is penned in and contained. Nothing in the chroot environment can see out past its own, special, root directory without escalating to root privileges. That has earned this type of environment the nickname of a chroot jail. 



使用 `chroot` ，可以在避免跟常规文件系统交互的前提下，设置并运行你的程序或者交互式 shell（sh, bash, zsh）.

`chroot` 环境中在没有升级到 root 权限时不能看到自己特殊的根目录。

这种环境被命名为 `chroot jail`.




## Creating a chroot Environment

We need a directory to act as the root directory of the chroot environment. So that we have a shorthand way of referring to that directory we’ll create a variable and store the name of the directory in it. Here we’re setting up a variable to store a path to the “testroot” directory. It doesn’t matter if this directory doesn’t exist yet, we’re going to create it soon. If the directory does exist, it should be empty.


我们需要一个目录来扮演 chroot 环境中的根目录。

这里创建一个变量用来存储该目录地址，这样我们可以以一种简短的方式来对其进行引用。


至于该目录目前不存在并没有关系，我们会在后面进行创建。如果目录确实存在也行，但是需要保证是一个全新的空目录。


```bash
chr=/home/linuxea/testroot
```

If the directory doesn’t exist, we need to create it. We can do that with this command. The -p (parents) option ensures any missing parent directories are created at the same time:

如果目录不存在，我们需要在这里开始进行创建。

我们可以使用这个命令

`-p` 选项是用来确保同时创建缺失的父目录

```bash
mkdir -p $chr
```


We need to create directories to hold the portions of the operating system our chroot environment will require. We’re going to set up a minimalist Linux environment that uses Bash as the interactive shell. We’ll also include the touch, rm, and ls commands. That will allow us to use all Bash’s built-in commands and touch, rm, and ls. We’ll be able to create, list and remove files, and use Bash. And—in this simple example—that’s all.



我们需要创建目录来保存我们的 `chroot` 环境对操作系统所需要的部分。

我们将配置并启动一个可以使用 bash 交互式 shell， 同时包含 `touch`, `rm`, `ls` 命令的最小化 linux 环境。

以使我们可以使用 bash 所有内置的命令以及额外的 touch， rm， ls 命令来创建，移除，列表文件。




Now we’ll change directory into our new root directory.

现在我们切换目录到我们新的根目录。

```bash
cd $chr
```


Let’s copy the binaries that we need in our minimalist Linux environment from your regular “/bin” directory into our chroot “/bin” directory. The -v (verbose) option makes cp tell us what it is doing as it performs each copy action.


我们将在最小 linux 环境下需要的二进制文件从常规文件系统的 `/bin` 目录复制到我们的 chroot `/bin` 目录。
`-v` 选项可以将 `cp` 命令执行明细打印出来
`--parents` 选项标记在目录下使用完整的文件名


```bash
cp -v --parents /bin/{bash,touch,ls,rm} $chr
```

The files are copied in for us:

复制的二进制文件：
```
/bin -> /home/linuxea/testroot/bin
'/bin/bash' -> '/home/linuxea/testroot/bin/bash'
'/bin/touch' -> '/home/linuxea/testroot/bin/touch'
'/bin/ls' -> '/home/linuxea/testroot/bin/ls'
'/bin/rm' -> '/home/linuxea/testroot/bin/rm'
```



These binaries will have dependencies. We need to discover what they are and copy those files into our environment as well, otherwise bash, touch, rm, and ls will not be able to function. We need to do this in turn for each of our chosen commands. We’ll do Bash first. The ldd command will list the dependencies for us.



这些二进制文本拥有自己的依赖项。

我们需要找出依赖并将它们同时复制到我们的最小化 linux 环境中，否则 `bash`, `touch`, `rm`, 与 `ls` 命令都将不能工作。

我们需要按顺序对选择的命令进行操作。

先从 `bash` 开始吧

`ldd` 会为我们列出二进制文件的依赖。

```bash
ldd /bin/bash
```

The dependencies are identified and listed in the terminal window:


依赖项被识别并在窗口终端上展示出来：

```bash
linuxea@linuxea-PC:~/testroot$ ldd /bin/bash
        linux-vdso.so.1 (0x00007fffe3498000)
        libtinfo.so.6 => /lib/x86_64-linux-gnu/libtinfo.so.6 (0x00007fc85a449000)
        libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007fc85a444000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fc85a283000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fc85a49b000)
```


We need to copy those files into our new environment. Picking the details out of that listing and copying them one at a time is going to be time-consuming and error-prone.


我们需要将这些文件拷贝到我们的最新环境。

从输出列表中选择出详情信息并一个个复制是耗时且易出错的。

Thankfully, we can semi-automate it. We’ll list the dependencies again, and this time we’ll form a list. Then we’ll loop through the list copying the files.


感谢我们可以将其半自动化。我们重新列出相关依赖项并形成一个列表，通过循环来进行复制操作。


Here we’re using ldd to list the dependencies and feed the results through a pipe into egrep. Using egrep is the same as using grep with the -E (extended regular expressions) option. The -o (only matching) option restricts the output to the matching parts of lines. We’re looking for matching library files that end in a number [0-9].



我们使用 `ldd` 列出依赖项并将结果通过管道作为 `egrep` 的输出。

`egrep` 命令的使用跟 `grep -E` 是相同的。

`-o` 选项用来约束输出来自匹配行中的一部分。

我们在寻找的以数字结尾的二进制文件。

```bash
list="$(ldd /bin/bash | egrep -o '/lib.*\.[0-9]')"
```




We can check the contents of the list using echo:

我们可以通过 `echo` 来检查 `list` 的内容：
```bash
echo $list
```

Now that we have the list, we can step through it with the following loop, copying the files one at a time. We’re using the variable i to step through the list. For each member of the list, we copy the file to our chroot root directory which is the value held in $chr.


现在我们拥有了文件列表，可以通过如下循环来逐步遍历，一次复制一个文件。

使用变量 `i` 来作为迭代列表的变量。

```bash
for i in $list; do cp -v --parent $i $chr; done
```


以下是输出结果：

/lib -> /home/linuxea/testroot/lib
/lib/x86_64-linux-gnu -> /home/linuxea/testroot/lib/x86_64-linux-gnu
'/lib/x86_64-linux-gnu/libtinfo.so.6' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libtinfo.so.6'
'/lib/x86_64-linux-gnu/libdl.so.2' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libdl.so.2'
'/lib/x86_64-linux-gnu/libc.so.6' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libc.so.6'
/lib64 -> /home/linuxea/testroot/lib64
'/lib64/ld-linux-x86-64.so.2' -> '/home/linuxea/testroot/lib64/ld-linux-x86-64.so.2'




We’ll use that technique to capture the dependencies of each of the other commands. And we’ll use the loop technique to perform the actual copying. The good news is we only need to make a tiny edit to the command that gathers the dependencies.



我们使用这种方式来捕获依赖项，通过循环来进行文件的复制操作。而且只需要少量的编辑就能够完成获取所有依赖的任务。


Here we’ve used the Up Arrow key to find the command, and we’ve edited it to say touch instead of bash.

这里我们使用键盘上面的箭头找回命令，并将编辑为 touch 而不是 bash。


```bash
list="$(ldd /bin/touch | egrep -o '/lib.*\.[0-9]')"
```

We can now repeat the exact same loop command as before:

现在重复执行与之前相同的循环操作：

```bash
 for i in $list; do cp -v --parent $i $chr; done
```


'/lib/x86_64-linux-gnu/libc.so.6' -> '/home/linuxea/testroot/lib/x86_64-linux-gnu/libc.so.6'
'/lib64/ld-linux-x86-64.so.2' -> '/home/linuxea/testroot/lib64/ld-linux-x86-64.so.2'






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



The last of our dependencies are copied into our chroot environment. 

We’re finally ready to use the chroot command. This command sets the root of the chroot environment, and specifies which application to run as the shell.


伴随着最后一个命令的依赖项复制到 chroot 环境的完成。

我们终于准备好使用 `chroot` 的准备工作。

此命令设置 `chroot` 环境的根目录，并指定作为 shell 运行的应用程序。

```bash
sudo chroot $chr /bin/bash
```


Our chroot environment is now active. The terminal window prompt has changed, and the interactive shell is the being handled by the bash shell in our environment.


我们的 `chroot` 环境现在处于活动状态。 

你可以注意终端窗口提示也已经发生了更改，同时交互式 shell 由我们环境中的 bash shell 处理。


We can try out the commands that we have brought into the environment.


我们可以使用引入到新环境的命令

```bash
bash-5.0# ls
bin  lib  lib64
```


The ls command works as we’d expect when we use it within the environment. When we try to access a directory outside of the environment, the command fails.


`ls` 命令在我们的新环境中如预料中工作。

当我们尝试访问环境外的目录时，命令发生失败
```bash
ls /home/linuxea/exists.txt
ls: cannot access '/home/linuxea/exists.txt': No such file or directory
```



We can use touch to create a file, ls to list it, and rm to remove it.


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

Of course, we can also use the built-in commands that the Bash shell provides. If you type help at the command line, Bash will list them for you.


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








Use exit to leave the chroot environment:


可以使用 `exit` 来退出 `chroot` 环境：
```bash
bash-5.0# exit
exit
linuxea@linuxea-PC:~/testroot$ 
```


If you want to remove the chroot environment, you can simply delete it:

如果你想要移除 `chroot` 环境， 你可以很简单地进行删除：

```bash
 rm -r $chr
```

## Automate for Convenience

If you’re thinking that chroot environments might be useful to you, but they’re a bit fiddly to set up, remember that you can always take the strain and the risk out of repetitive tasks by using aliases, functions, and scripts.


通过这篇文章下来，如果你觉得 `chroot` 对你是有用的，但设置起来有点麻烦，记住你总是能够通过别名，函数，脚本的形式来消除重复性的工作。
而不要因此停止对 `chroot` 的探索。




































