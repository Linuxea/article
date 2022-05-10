# Customing Go Binaries with Build Tags


## Introduction

In Go, a build tag, or a build constraint, is an identifier added to a piece of code that determines when the file should be included in a package during the build process. This allows you to build different versions of your Go application from the same source code and to toggle between them in a fast and organized manner. Many developers use build tags to improve the workflow of building cross-platform compatible applications, such as programs that require code changes to account for variances between different operating systems. Build tags are also used for integration testing, allowing you to quickly switch between the integrated code and the code with a mock service or stub, and for differing levels of feature sets within an application.


Go 的构建标签决定文件是否引入构建过程。

这个功能可以使开发者从是同一份源码中构建不同的应用版本，并且是以一种快速且有组织的方式。

许多开发者使用构建标签来提升跨平台兼容应用构建的工作流。
比如变更代码来解决不同的操作系统间的差异。

Let’s take the problem of differing customer feature sets as an example. When writing some applications, you may want to control which features to include in the binary, such as an application that offers Free, Pro, and Enterprise levels. As the customer increases their subscription level in these applications, more features become unlocked and available. To solve this problem, you could maintain separate projects and try to keep them in sync with each other through the use of import statements. While this approach would work, over time it would become tedious and error prone. An alternative approach would be to use build tags.

本方我们以一个不同客户功能集合作为例子。

在写一些应用程序时，你可能想要控制哪一些功能来引入到二进制可执行文件中，比如一个应用提供了 Free, Pro 与 Enterprise 不同级别，随着用户提升订阅等级，就能够解锁更多的功能使其可用。

为了解决这个问题，你可以维护不同的项目并且通过导入语句来保持它们之间的同步，这虽然可行，然而随着时间推移会变得冗余与易错。

另外一个可行的方案是使用构建标签。

In this article, you will use build tags in Go to generate different executable binaries that offer Free, Pro, and Enterprise feature sets of a sample application. Each will have a different set of features available, with the Free version being the default.


在本方中，你将会在 Go 中使用构建标签来生成不同的可执行二进制文件，分别提供了 Free, Pro 与 Enterprise 功能集合，Free 是默认的。




## Prerequisites

