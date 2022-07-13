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

```