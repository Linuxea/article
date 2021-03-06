## Git Submodules


> How to make multiple repositories work together.

`Git Submodules` 已经存在很久了，但是依然有众多开发者从未使用过。

得益于今日众多的包管理与共享共享库的选项，git submodules 并不适用于每个你所需要的共享库或者依赖的场景。

Git submodules 主要应用在当你想要从源与消费角度对代码进行更强的控制。

Git submodules 简单来讲只是对另外仓库的引用。

你可以拥有一个集成项目 `sdk`，同时依赖于多个子项目 `compile`, `runtime` 等。

集成项目作为主要目录，而多个子项目作为模块。

如果你想要同时对 sdk 及子项目作为变更，那么 git submodules 就能够作为一个备选项。


本文会提到 git submodules 的关键特性，包含如何创建，基本功能，以及最后会提到主要亮点总结。


### Adding Git Submodules to a New Project

我们从创建一个简单的项目 `sdk` 开始，它包含一个 `README.md` 文件以及 `git` 目录。

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

我们在远程 GitHub 已经准备了另外一个现成仓库，现在我们可以轻易地向 sdk 仓库添加此子模块：
```zsh
➜  sdk git:(master) git submodule add git@github.com:Linuxea/compiler.git        
Cloning into '/home/linuxea/sdk/compiler'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 3 (delta 0), pack-reused 0
Receiving objects: 100% (3/3), done.
```

现在我们在 sdk 仓库下拥有 `compiler` 仓库以及 `.submodules` 文件。

`.submodules` 仅仅记录了我们所拥有的子模块的基本信息，如：名称，路径与 url 等。
```zsh
➜  sdk git:(master) cat .gitmodules 
[submodule "compiler"]
	path = compiler
	url = git@github.com:Linuxea/compiler.git
```

当我们拥有多个子模块时，相应的就会存在对应的多个配置。

在重新得到一个干净的 git 状态之前，我们需要 `add` 和 `commit` 这些新文件。
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

打印出来的是 `compiler` 子模块的主分支上最新的一个 `commit`.

通过这种方式，git submodules 只是作为从一个存储库到下一个存储库的链接，具有额外的好处是每个存储库的源代码都易于访问和共享。


### Working with Git Submodules

根据默认配置，任何基本的 git 命令仅在当前所在的仓库环境下运行。

所以假如你在 sdk 目录之下（compiler 目录之外）作出更改，git status 仅仅会展示 sdk 的不同, 同理 git pull, push, commit 等等仅针对你当前所在的存储库执行操作。

如果你想要在项目中的所有子模块运行命令，例如 git pull, 你可以从 sdk 运行 `git pull --recurse-submodules`.

这将拉取对 sdk 以及从每个子模块当前所在的分支中拉取新的更改。

如果子模块也拥有其子模块，这个过程会一直进行直到完成所有的更新。

为了命令的简洁以及自动拉取子模块，你可以通过运行命令 `git config submodule.recurse true` 来更改配置。
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

为了在子模块运行常规命令，使用 `foreach` 迭代：
```zsh
➜  sdk git:(master) git submodule foreach ls
Entering 'compiler'
README.md
➜  sdk git:(master) 
```


### Tracking submodule commit references

现在我们对 sdk 的一个子模块做出变更，添加以及提交，然后回到 sdk 根目录下运行 git status, 你会看到如下信息：
```zsh
➜  sdk git:(master) cd compiler 
➜  compiler git:(master) touch JIT   
➜  compiler git:(master) ✗ git add .
➜  compiler git:(master) ✗ git commit -m "add JIT" 
[master 185844e] add JIT
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 JIT
➜  compiler git:(master) cd ..
➜  sdk git:(master) ✗ git status
On branch master
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   compiler (new commits)

no changes added to commit (use "git add" and/or "git commit -a")
```

Git 现在告诉你的是当前 sdk 的子模块状态变更了，你需要添加子模块的变更到 git 追踪并进行提交。
```zsh
➜  sdk git:(master) ✗ git add .
➜  sdk git:(master) ✗ git commit -m "update compiler to add JIT"
[master bb53126] update compiler to add JIT
 1 file changed, 1 insertion(+), 1 deletion(-)
➜  sdk git:(master) git status
On branch master
nothing to commit, working tree clean
```


### Cloning a project including its submodules

当你拉取一个拥有子模块的仓库时，默认是不会拉取子模块目录。你得到除了目录引用，除此之外没有额外内容。
```zsh
➜  temp git clone git@github.com:Linuxea/sdk.git
Cloning into 'sdk'...
remote: Enumerating objects: 8, done.
remote: Counting objects: 100% (8/8), done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 8 (delta 1), reused 8 (delta 1), pack-reused 0
Receiving objects: 100% (8/8), done.
Resolving deltas: 100% (1/1), done.
➜  temp cd sdk 
➜  sdk git:(master) ls
compiler  README.md
➜  sdk git:(master) ls -alh compiler 
total 8.0K
drwxrwxr-x 2 linuxea linuxea 4.0K Jul 14 01:00 .
drwxrwxr-x 4 linuxea linuxea 4.0K Jul 14 01:00 ..
```

为了克隆包括子模块所有文件的项目仓库
- 你可以使用 `--recurse-submodules` 标识，
- 可以在克隆 sdk 后运行 `git submodule update --init`

以上两种方式都能实现拉取所有关联文件的任务。

紧接上面的实验，我们使用第二种方式:
```zsh
➜  sdk git:(master) ls -alh compiler
total 8.0K
drwxrwxr-x 2 linuxea linuxea 4.0K Jul 14 01:00 .
drwxrwxr-x 4 linuxea linuxea 4.0K Jul 14 01:00 ..
➜  sdk git:(master) 
➜  sdk git:(master) git submodule update --init 
Submodule 'compiler' (git@github.com:Linuxea/compiler.git) registered for path 'compiler'
Cloning into '/home/linuxea/temp/sdk/compiler'...
Submodule path 'compiler': checked out '185844e0670c65b13b7ebc06353bb301c0ac0e3f'
➜  sdk git:(master) 
➜  sdk git:(master) ls -alh compiler
total 12K
drwxrwxr-x 2 linuxea linuxea 4.0K Jul 14 01:04 .
drwxrwxr-x 4 linuxea linuxea 4.0K Jul 14 01:00 ..
-rw-rw-r-- 1 linuxea linuxea   33 Jul 14 01:04 .git
-rw-rw-r-- 1 linuxea linuxea    0 Jul 14 01:04 JIT
-rw-rw-r-- 1 linuxea linuxea    0 Jul 14 01:04 README.md
```

### Git Submodules Highlights

如你所见的是，git submodules 有别于常规的包管理。

它允许你链接其他仓库，但仍然可视它们为个体独立的项目。

事实上 Git Submodules 在过去的时间中收到了一个公平的正向反馈，不过更重要的是客观地看待 git submodules 以及它是否适用于你将来的项目。

- 在 git 仓库之间建立链接来容易访问
- 跨越多个子模块来同时执行命令
- 精准且密切维护仓库版本
- 将多个仓库视为一个分发

#### 需要更多的考虑
- 引入额外的版本控制复杂度


## 参考

- [1] [7.11 Git Tools - Submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules)
- [2] [An Intro to Git-Submodules](https://medium.com/gitconnected/an-intro-to-git-submodules-9f49523281fb)