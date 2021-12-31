
> 本文解释 linux 上 apt 与 apt-get 命令之间的区别。同时也会列举一些替代 apt-get 的最常用命令


apt 第一个稳定版本是在 2014 发布，不过直到 2016 年的时候才开始被大多数人注意到。

使用 apt 取代 apt-get 变得更常见。最终，许多其他的发行厂商都跟随 ubuntu 的脚步，鼓励用户使用 apt 而不是 apt-get 

你可能会好奇 apt 与 apt-get 之间的区别是什么？和它们之间是否有相似的命令结构？
新的 apt 命令的需求场景是什么？你可能也会思考是否 apt 比 apt-get 更好？
而我们应该使用新的 apt 命令还是坚持历久弥香的 apt-get 命令？

因此我将在本文回答关于上面的这些问题，同时在文章最后，我希望你们有幅清晰的画像。


## apt vs apt-get
这里快速地给 linux mint 用户说几句话。几年前，linux mint 实现了一个命令 apt，实际上是用 python 对 apt-get 的包装，同时提供了更多的友好选项。
本文我们讨论的 apt 跟 linux mint 的 apt 是不同的。




## 为什么把引入 apt 放在首位
Debian，作为 ubuntu，linux mint，elementary OS 等的 linux 发行版母版，拥有强大的包管理系统。
每个组件与应用都以包的形式安装在你的系统。Debian 使用一系列名为 Advanced Packaging Tool(APT) 的工具来管理包系统。注意这里不要与命令 apt 混合在一起造成困惑。它们是不一样的。

在基于 Debian 的 linux 发行版上，有许多通过与 APT 交互的工具允许你安装，删除卸载，管理包。
apt-get 就是其中广泛使用的一种。另外一个流行的工具是兼具GUI 与命令行的 Aptitude。

如果你看过我关于 apt-get 命令指南，你可能会遇到许多类似的命令，比如 apt-cache。而这就是问题所在。

你看，这些命令都太底层，并且拥有大部份用户永远不会使用到的功能。另外一方面，常用的命令却散落在 apt-get 与 apt-cache中。


apt 正是引入来解决这些问题。
apt 坚持了整合来自 apt-get 与 apt-cache 的部分命令，同时摒弃了晦涩且很少使用的特性。同时它也可以管理 apt.conf 文件。


使用 apt，你不必在 apt-get 与 apt-cache 之间来回摆弄。apt 更加结构化，并为您提供管理软件包所需的必要选项。


## apt 与 apt-get 的区别

因此使用 apt 就可以在一个地方获得所有必要的工具，而不会迷失在无数的命令行选项中。

apt 的主要目标是以一种“愉悦终端用户”的方式来提供更有效率的管理包的办法。

在 Debian 提到“愉悦终端用户”，指的是以更好的方式组织少而效率的命令和选项。其中最重要的也是最能说明的是，它开启一些对终端用户真正有用的选项作为默认。

比如，使用 apt 你可以看得到安装过程中的进度提示。

apt 会在更新仓库提示你有多少软件可以更新升级。

使用 apt-get 结合命令行选项也可以实现这一点，而 apt 将它作为无痛的默认行为。


## apt 与 apt-get 命令的区别

尽管 apt 与 apt-get 有着一些类似的命令行选项，但是这不意味着 apt 向后兼容 apt-get。
这意味着如果只是简单地把 apt-get 替换成 apt 并不问题奏效。

我们可以看下 apt 用来替换 apt-get 与 apt-cache 的一些命令。

| apt command | the command it replaces | function of the command |
| ----------  | ----------------------  | ----------------------  |
| apt install | apt-get install | install a package |
| apt remove | apt-get remove | remove a package |
| apt purge | apt-get purge | remove a package with configuration |
| apt update | apt-get update | refresh repository index |
| apt upgrade | apt-get upgrade | upgrades all upgradable packages |
| apt autoremove | apt-get autoremove | remove unwanted packages |
| apt full-upgrade | apt-get disk-upgrade | upgrade packages with auto-handling of dependencies |
| apt search | apt-cache search | search for the program | 
| apt show | apt-cache show | show package details |


apt 还有一些自己特有的命令
| new apt command | function of the command |
| ------------- | ----------------- |
| apt list | list packages with criteria(install upgradable etc) |
| apt edit-sources | edit source list | 


有一点需要提到的是，apt 命令还在持续发展中。所以你可能会看到新的命令将出来未来的版本里。


## apt-get 废弃了吗？

我没有看到任何关于 apt-get 将停产的消息。并且这也不应该发生。它依然提供比 apt 多很多的功能。


## 我该选择 apt 还是 apt-get？

你可能会思考你应该使用 apt 还是 apt-get。我的答案是，作为一个普通 linux 用户，应该使用的是 apt。

apt 是作为被 linux 发行商推荐的命令。它提供了必要的选择来管理包。最重要的一点它更少更容易记得参数。

我看不到关于继续使用 apt-get 的理由，除非你执行的是关于 apt-get 的特定操作们。


## 总结
我希望我能解释清晰关于 apt 与 apt-get 之间的区别。在最后我总结关于 apt 与 apt-get 的一些争议：
- apt 是 apt-get 与 apt-cache 的功能子集，却提供了必要的包管理命令
- 尽管 apt-get 不会被淘汰，但是作为一个普通的用户，你应该开始经常使用 apt








































































