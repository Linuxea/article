# Java 的编程式 Debug 

在惊叹 Intellij IDEA 与 Eclipse 等集成开发工具的调试功能时，这个问题我们可能都会好奇过这是如何实现的。
而这在很大程度上是依赖于 `Java Platofrom Debugger Architecture(JPDA)` Java 平台调试架构。





## JDPA

JPDA 是 Java Platform Debugger Architecture 的缩写，它是用于在 Java 平台上进行调试的架构。JPDA 提供了一组 API，允许开发人员编写调试器，以便在运行时检查和控制 Java 应用程序的执行。JPDA 由三个组件组成：
- Java 虚拟机工具接口（JVMTI）帮助我们与在JVM中运行的应用程序进行交互和控制执行。
- Java 调试线协议（JDWP）定义了在测试中的应用程序（被调试者）和调试器之间使用的协议。
- Java 调试接口（JDI）用于实现调试器应用程序。


## JDI

Java调试接口API是Java提供的一组接口，用于实现调试器的前端。JDI是JPDA的最高层。

它提供了访问VM及其状态以及访问被调试程序的变量的能力。同时，它允许设置断点、单步执行、监视点和处理线程。


JDI 的术语有以下：
- Debugger program ：调试程序，用来调试其他程序的程序
- Debug target program: 被调试的目标程序
- Virtual Machine(VM)：运行目标程序的虚拟机
- Connector：调试程序与目标虚拟机的连接器
- Events：当虚拟机以调试模式运行目标程序时，它会触发多种事件以便调试程序采取处理。调试程序也可以请求虚拟机触发特定事件来应对默认不触发的场景

