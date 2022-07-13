## Git Submodules


> How to make multiple repositories work together.

Git submodules have been around for more than a decade, and yet many developers have never used them. Thanks to today’s list of options for package managers and shared libraries, git submodules aren’t meant for every scenario where you need to share packages or dependencies. Submodules are primarily meant to be used when you want more control on changing code both from a source and consumer standpoint. Let’s expand on that idea.


Git submodules 已经存在很久了，但是依然有众多开发者从未使用过。

得益于今日众多的包管理与共享共享库的选项，git submodules 并不适用于每个你所需要的共享库或者依赖的场景。

Git submodules 主要应用在当你想要从源与消费角度对代码进行更强的控制。

First off, a git submodule is simply a reference to another repository. You could have an overarching project, say your own SDK (Software Development Kit). That SDK could depend on sub-projects such as a compiler, a runtime to handle execution of compiled binaries, code samples and more. The SDK would be your main project repository, with the compiler and runtime as submodules (of course you would need more sub-module projects than just these two in a real application). If you need to be making changes in the SDK source code and its sub-projects at once, git submodules could be an option for you.


Git submodules 简单来讲只是对另外仓库的引用。

你可以拥有一个集成项目 sdk，同时依赖于多个子项目 compile, runtime 等。

集成项目作为主要目录，而多个子项目作为模块。

如果你想要同时对 sdk 及子项目作为变更，那么 git submodules 可以在此时作为一个工具选项。


This article will feature some of the key characteristics of git submodules including how to create them, basic functionality as well as a summary of their key highlights at the end.


本文会提到 git submodules 的关键特性，包含如何创建，基本功能，以及最后会提到主要亮点总结。



### Adding Git Submodules to a New Project

It’s easy enough to add submodules to an existing git repository, but there are a few things to be aware of. 

Let’s start off with creating a simple project sdk that has the single file README.md as well as git tracking.

我们从创建一个简单的项目 sdk 开始，它包含一个 README.md 文件以及 git 目录。

```zsh
➜  ~ pwd
/home/linuxea
➜  ~ mkdir sdk
➜  ~ cd sdk 
➜  sdk git init
Initialized empty Git repository in /home/linuxea/sdk/.git/
➜  sdk git:(master) touch README.md 
➜  sdk git:(master) ✗ git add .
➜  sdk git:(master) ✗ git commit -m "init sdk project"
[master (root-commit) 2693bab] init sdk project
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 README.md
```


Now, let’s say that we have another repository that already has a remote origin in GitHub. We can easily add the submodule to our sdk repo like so:

我们在远程 GitHub 已经准备了另外一个现成仓库，现在我们可以轻易地向 sdk 仓库添加此子模块：
```zsh
➜  sdk git:(master) git submodule add git@github.com:Linuxea/compiler.git        
Cloning into '/home/linuxea/sdk/compiler'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 3 (delta 0), pack-reused 0
Receiving objects: 100% (3/3), done.
```

We will now have the compiler repository within sdk along with the .gitmodules file. This file just has basic information on our submodules such as their name, path, url etc. You’ll need to add and commit these new files before reaching a clean git state again. Checking the state of our submodules is easy.

现在我们在 sdk 仓库下拥有 compiler 仓库以及 .submodules 文件。

.submodules 仅仅记录了我们所拥有的子模块的基本信息，如：名称，路径与 url 等。

在重新得到一个干净的 git 状态之前，我们需要 add 和 commit 这些新文件。
```zsh
➜  sdk git:(master) ✗ git status
On branch master
Changes to be committed:
  (use "git restore --staged <file>..." to unstage)
	new file:   .gitmodules
	new file:   compiler

➜  sdk git:(master) ✗ git commit -m "add submodule compiler"
[master 883ee91] add submodule compiler
 2 files changed, 4 insertions(+)
 create mode 100644 .gitmodules
 create mode 160000 compiler
```

检查子模块的状态是简单的：
```zsh
➜  sdk git:(master) git submodule status
 482fb8b4768049cfae8726ce3f4f7bf7cbe00776 compiler (heads/master)
```

The commit reference that’s listed for compiler is the most recent commit for that repository’s primary branch. In this way, git submodules are just links from one repository to the next with the added benefit that the source code for each repository is easily accessed and shared.

打印出来的是 compiler 子模块的主分支上最新的一个 commit.

通过这种方式，git submodules 只是从一个存储库到下一个存储库的链接，具有额外的好处是每个存储库的源代码都易于访问和共享。


### Working with Git Submodules

By default, any basic git commands are only ran with respect to the repository that you are currently in. So if you make changes in the sdk repo (but outside the compiler repo), then git will only show you those differences in sdk. Pulling, pushing, committing etc. are all performed only with regards to the repo you are currently in.

根据默认配置，任何基本的 git 命令仅在当前所在的仓库环境下运行。

所以假如你在 sdk 目录之下（compiler 目录之外）作出更改，git status 仅仅会展示 sdk 的不同, 同理 git pull, push, commit 等等仅针对你当前所在的存储库执行操作。

If you want to run a command such as git pull across all of the submodules in your project, you‘d run git pull --recurse-submodules from sdk. This would pull any changes made to sdk as well as pulling new changes from the branch each of your submodules are currently on. If your submodules had submodules, the process would continue until all are updated. To automatically pull submodules whenever you run git pull, you can run git config submodule.recurse true.

如果你想要在项目中的所有子模块运行命令，例如 git pull, 你可以从 sdk 运行 `git pull --recurse-submodules`.

这将拉取对 sdk 以及从每个子模块当前所在的分支中拉取新的更改。

如果子模块也拥有其子模块，这个过程会一直进行直到完成所有的更新。

为了命令的简洁以及自动拉取子模块，你可以通过运行命令 git config submodule.recurse true 来更改配置。
```zsh
➜  sdk git:(master) git config submodule.recurse true 
➜  sdk git:(master) git config -l | cat
user.name=linuxea
user.email=linuxea@foxmail.com
core.repositoryformatversion=0
core.filemode=true
core.bare=false
core.logallrefupdates=true
submodule.compiler.url=git@github.com:Linuxea/compiler.git
submodule.compiler.active=true
submodule.recurse=true
➜  sdk git:(master) git pull
Fetching submodule compiler
... ...
```


For running more general commands across your submodules, use the foreach term to loop over each of them. Example:

为了在子模块运行常规命令，使用 foreach 迭代：
```zsh
➜  sdk git:(master) git submodule foreach ls
Entering 'compiler'
README.md
➜  sdk git:(master) 
```



### Tracking submodule commit references