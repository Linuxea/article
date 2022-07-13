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







