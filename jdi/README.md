# Java 的编程式 Debug 

在惊叹 Intellij IDEA 与 Eclipse 等集成开发工具的调试功能时，我们可能都会好奇过这是如何实现的。
而这在很大程度上是依赖于 `Java Platofrom Debugger Architecture(JPDA)`  Java 平台调试架构。


## JDPA

JPDA 是 Java Platform Debugger Architecture 的缩写，它是用于在 Java 平台上进行调试的架构。JPDA 提供了一组 API，允许开发人员编写调试器，以便在运行时检查和控制 Java 应用程序的执行。

JPDA 由三个组件组成：
- Java 虚拟机工具接口（JVMTI）帮助我们与在JVM中运行的应用程序进行交互和控制执行。
- Java 调试线协议（JDWP）定义了在测试中的应用程序（被调试者）和调试器之间使用的协议。
- Java 调试接口（JDI）用于实现调试器应用程序。

## JDI

Java调试接口API是Java提供的一组接口，用于实现调试器的前端。JDI 是 JPDA 的最高层。

它提供了访问 VM 及其状态以及访问被调试程序的变量的能力。同时，它允许设置断点、单步执行、监视点和处理线程。

JDI 的术语有以下：
- `Debugger program` ：调试程序，用来调试其他程序的程序，比如 IDEA 提供的调试功能。
- `Debug target program`: 被调试的目标程序，比如我们的日常代码。
- `Virtual Machine(VM)`：运行目标程序的虚拟机。
- `Connector`：调试程序与目标虚拟机的连接器。
- `Events`：当虚拟机以调试模式运行目标程序时，它会触发多种事件以便调试程序采取处理。调试程序也可以请求虚拟机触发特定事件来应对默认不触发的场景。

## Objective

今天我们会通过编写一些案例来实现编程式的调试，通过这些步骤来熟悉 `JDI`。

## Setup

首先，我们需要两个独立的程序，`debuggee` 与 `debugger`。

让我们创建一个名为 `JDIExampleDebuggee` 的类，其中包含几个字符串变量和打印语句，作为一个简单的被调试程序。
```java
package com.linuxea;

public class JDIExampleDebuggee {

  public static void main(String[] args) {
    String jpda = "Java Platform Debugger Architecture";
    System.out.println("Hi Everyone, Welcome to " + jpda); // add a break point here

    String jdi = "Java Debug Interface"; // add a break point here and also stepping in here
    String text = "Today, we'll dive into " + jdi;
    System.out.println(text);
  }
}
```


让创建一个名为 `JDIExampleDebugger` 的类，其中包含用于保存调试程序（debugClass）和断点行号（breakPointLines）的属性：
```java
public class JDIExampleDebugger<T> {

  private Class<T> debugClass; //作为被调试程序
  private int[] breakPointLines; //自定义设置的多个断点行数

  // 待完善
}
```

### LaunchingConnector

Debugger 程序首先需要一个连接器来连接目标程序的虚拟机，然后我们需要把 debuggee 作为设置连接器的 `main` 参数，最后连接器会启动虚拟机来开始调试。

为此，JDI 提供了一个 `Bootstrap` 类，该类提供了一个 `LaunchingConnector` 实例。LaunchingConnector 提供了默认参数的映射，我们可以在其中设置主参数。


因此，让我们将 `connectAndLaunchVM` 方法添加到 `JDIExampleDebugger` 类中：
```java
public VirtualMachine connectAndLaunchVM() throws Exception {

  LaunchingConnector launchingConnector = Bootstrap.virtualMachineManager().defaultConnector();
  Map<String, Argument> arguments = launchingConnector.defaultArguments();
  arguments.get("main").setValue(debugClass.getName());

//If you run this program as part of maven project
// then classes will be compiled in target directory.
//So you might need below additional env argument in code of JDIExampleDebuggee.
Connector.Argument options = arguments.get("options");
options.setValue("-cp \"C:\\Users\\Linux\\Desktop\\code\\jdi\\target\\classes\"");

return launchingConnector.launch(arguments);
}
```


现在我们在 `JDIExampleDebugger` 添加 `main` 方法来进行调试：
```java
public static void main(String[] args) {

  JDIExampleDebugger<JDIExampleDebuggee> debuggerInstance = new JDIExampleDebugger<>();
  debuggerInstance.debugClass = JDIExampleDebuggee.class;
  debuggerInstance.breakPointLines = new int[]{6}; //设置一个断点行数为第6行
  VirtualMachine vm; //目标虚拟机对象
  try {
    vm = debuggerInstance.connectAndLaunchVM(); //启动并连接目标虚拟机
    vm.resume(); // 虚拟机进行运行
  } catch (VMDisconnectedException e) {
    System.out.println("Virtual Machine is disconnected.");
  } catch (Exception e) {
    e.printStackTrace();
  }
}
```

> jdi 相关类并不在标准包中，需要在 `classpath` 目录下添加 `tools.jar`


## Bootstrap and ClassPrepareRequest

目前为止 debugger 还没有真正的实现编程式调试，因为我们还没有完成调试类的`准备工作`与`断点设置`。

`VirtualMachine` 类提供了一个 `eventRequestManager` 方法，来创建多种不同类型的请求，如：`ClassPrepareRequest`，`BreakpointRequest` 以及 `StepEventRequest`。

添加 `enableClassPrepareRequest` 来开启指定类的准备工作：
```java
public void enableClassPrepareRequest(VirtualMachine vm) {
  ClassPrepareRequest classPrepareRequest = vm.eventRequestManager().createClassPrepareRequest();
  classPrepareRequest.addClassFilter(debugClass.getName());
  classPrepareRequest.enable();
}
```

## ClassPrepareEvent and BreakpointRequest

一旦开启类的准备工作请求，`ClassPrepareEvent` 事件实例将会在事件队列中出现。

使用 ClassPrepareEvent，我们能够获取位置来设置断点并且创建 `BreakPointRequest`。

```java
public void setBreakPoints(VirtualMachine vm, ClassPrepareEvent event)
    throws AbsentInformationException {
  ClassType classType = (ClassType) event.referenceType();
  for (int lineNumber : breakPointLines) {
    Location location = classType.locationsOfLine(lineNumber).get(0);
    BreakpointRequest bpReq = vm.eventRequestManager().createBreakpointRequest(location);
    bpReq.enable();
  }
}
```

## BreakPointEvent and StackFrame

目前为此，我们完成了类准备与设置断点的工作。

现在我们需要捕捉断点并且打印断点位置的变量。
```java
public void displayVariables(LocatableEvent event) throws IncompatibleThreadStateException,
    AbsentInformationException {
StackFrame stackFrame = event.thread().frame(0);
  if (stackFrame.location().toString().contains(debugClass.getName())) {
    Map<LocalVariable, Value> visibleVariables = stackFrame
        .getValues(stackFrame.visibleVariables());
    System.out.println("Variables at " + stackFrame.location().toString() + " > ");
    for (Map.Entry<LocalVariable, Value> entry : visibleVariables.entrySet()) {
    System.out.println(entry.getKey().name() + " = " + entry.getValue());
    }
  }
}
```

## StepRequest

调试还需要逐步执行代码并检查后续步骤中变量的状态。因此，在断点处再创建一个 `step` 调试请求。
```java
public void enableStepRequest(VirtualMachine vm, BreakpointEvent event) {
  // enable step request for last break point
  if (event.location().toString().contains(debugClass.getName() + ":" + breakPointLines[breakPointLines.length - 1])) {
    StepRequest stepRequest = vm.eventRequestManager()
        .createStepRequest(event.thread(), StepRequest.STEP_LINE, StepRequest.STEP_OVER);
    stepRequest.enable();
  }
}
```

## Debug Target

现在我们在 Debugger 的 main 方法中整合 `enableClassPrepareRequest`, `setBreakPoints`, 与 `displayVariables`:
```java
public static void main(String[] args) {

  JDIExampleDebugger<JDIExampleDebuggee> debuggerInstance = new JDIExampleDebugger<>();
  debuggerInstance.debugClass = JDIExampleDebuggee.class;
  debuggerInstance.breakPointLines = new int[]{6};
  VirtualMachine vm;
  try {
    vm = debuggerInstance.connectAndLaunchVM();
    debuggerInstance.enableClassPrepareRequest(vm);
    EventSet eventSet;
    while ((eventSet = vm.eventQueue().remove()) != null) {
    for (Event event : eventSet) {
      System.out.println(event.getClass().getName());

      if (event instanceof ClassPrepareEvent) {
      debuggerInstance.setBreakPoints(vm, (ClassPrepareEvent) event);
      }
      if (event instanceof BreakpointEvent) {
      debuggerInstance.displayVariables((BreakpointEvent) event);
      }

      if (event instanceof BreakpointEvent) {
      debuggerInstance.enableStepRequest(vm, (BreakpointEvent) event);
      }

      if (event instanceof StepEvent) {
      debuggerInstance.displayVariables((StepEvent) event);
      }
      vm.resume();
      }
    }
  } catch (VMDisconnectedException e) {
    System.out.println("Virtual Machine is disconnected.");
  } catch (Exception e) {
    e.printStackTrace();
  }
}
```

运行结果如下：
```
Variables at com.linuxea.JDIExampleDebuggee:6 > 
args = instance of java.lang.String[0] (id=76)
Variables at com.linuxea.JDIExampleDebuggee:7 > 
jpda = "Java Platform Debugger Architecture"
args = instance of java.lang.String[0] (id=76)
Variables at com.linuxea.JDIExampleDebuggee:9 > 
jpda = "Java Platform Debugger Architecture"
args = instance of java.lang.String[0] (id=76)
Variables at com.linuxea.JDIExampleDebuggee:10 > 
jdi = "Java Debug Interface"
jpda = "Java Platform Debugger Architecture"
args = instance of java.lang.String[0] (id=76)
Variables at com.linuxea.JDIExampleDebuggee:11 > 
text = "Today, we'll dive into Java Debug Interface"
jdi = "Java Debug Interface"
jpda = "Java Platform Debugger Architecture"
args = instance of java.lang.String[0] (id=76)
Variables at com.linuxea.JDIExampleDebuggee:12 > 
text = "Today, we'll dive into Java Debug Interface"
jdi = "Java Debug Interface"
jpda = "Java Platform Debugger Architecture"
args = instance of java.lang.String[0] (id=76)
Virtual Machine is disconnected.

Process finished with exit code 0
```

我们注意到了 JDIExampleDebuggee 程序中的 `print` 结果没有出现在调试器的输出结果中。

根据 `JDI` 文档，如果我们通过 `LaunchingConnector` 启动 `VM`，则必须通过 `Process` 对象读取其输出和错误流。
因此，让我们将其添加到主方法的 `finally` 子句中：
```java
finally {
    InputStreamReader reader = new InputStreamReader(vm.process().getInputStream());
    OutputStreamWriter writer = new OutputStreamWriter(System.out);
    char[] buf = new char[512];
    int read = reader.read(buf);
    writer.write(buf, 0, read);
    writer.flush();
}
```

打印如下：
```java
Hi Everyone, Welcome to Java Platform Debugger Architecture
Today, we'll dive into Java Debug Interface
```



## Extension - Remote Debug

除了本地编程调试，还想再提一下 java 的远程调试，在本地调试的基础上，添加了网络模块，实现了远程调试。

Java 远程调试是指在远程计算机上运行的 Java 应用程序的调试过程。它允许开发人员在远程计算机上调试 Java 应用程序，就好像它们在本地计算机上运行一样。Java 远程调试需要使用 Java Platform Debugger Architecture（JPDA）来建立调试会话。调试会话包括远程计算机上的应用程序和本地计算机上的调试器之间的通信。为了进行远程调试，需要在远程计算机上启动应用程序，并将其连接到本地计算机上运行的调试器。这样，开发人员就可以在本地计算机上检查和控制远程计算机上运行的应用程序的执行。

举个粟子,以我们熟悉的 spring boot api 为例:
```java
@RestController
public class HelloController {

  @GetMapping("/hello")
  public String hello(@RequestParam String name) {
    System.out.println(name);
    return "hello:" + name;
  }
}
```

打包部署到远程服务器上并运行:
```bash
mvn clean install package
```

上传到远程服务器并运行
```bash
➜  ~ cd ~/remote-debug 
➜  remote-debug /root/dev/jdk-20.0.1/bin/java '-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005' -jar remotedebug-0.0.1-SNAPSHOT.jar
```

这是一个启动一个 Java 应用程序并开启远程调试的命令。

具体来说，它启动了一个名为 `remotedebug-0.0.1-SNAPSHOT.jar` 的 Java 应用程序，并将其连接到一个监听端口为 5005 的调试器。这个命令中的 `-agentlib:jdwp` 参数告诉 JVM 启用 JPDA 调试代理，`transport=dt_socket` 参数指定了使用 socket 进行通信，`server=y` 参数表示 JVM 将充当调试服务器，`suspend=n` 参数表示启动 JVM 后不暂停，`address=*:5005` 参数指定了监听的地址和端口号。

这样，开发人员就可以使用一个支持 JDWP 协议的调试器连接到这个端口，并远程调试这个 Java 应用程序了。

在支持远程调试的 IDE 上如 Intellij IDEA 上开启远程调试功能:

![remote-debug.png](remote-debug.png 'remote-debug.png')


启动配置好的远程 debug 配置，运行如图：

![remote-debug1.png](remote-debug1.png 'remote-debug1.png')


向远程服务器发送请求：
```bash
➜  ~ curl http://119.91.226.57:8080/hello\?name\=linuxea
```

本地调试程序收到请求并完成对程序的调试：
![remote-debug2.png](remote-debug2.png 'remote-debug2.png')

远程调试的好处不言而喻，特别是对于本地调试不太方便的场景：
1. 更快的故障排除：如果在更新线上服务时出现问题，Java 远程调试可以帮助开发人员更快地诊断和解决问题。开发人员可以在应用程序出现问题时立即进行调试，而不必等待应用程序停止或重新启动。

2. 更高效的更新：Java 远程调试可以加快更新速度，因为它允许开发人员在本地计算机上进行调试，而不必在远程计算机上进行调试。这样可以节省时间和精力，使开发人员更专注于代码的编写和测试。

3. 更高效的团队协作：Java 远程调试可以帮助团队更高效地协作。开发人员可以在本地计算机上进行调试，并将结果共享给其他团队成员，以便他们可以更快地了解应用程序的状态和进展。这样可以提高团队的协作效率，使团队更加紧密和协调。

4. ... ... 等等



## Conclusion

这篇文章中，我们对 JPDA 中的 JDI 进行了初步的探索，了解了 java 世界探索的初步奥秘。同时我们简要介绍了与此息息相关的远程调试，并提供了实例详解。


## Reference

- Java Debug Interface API (JDI) – Hello World Example | Programmatic debugging for beginners: https://itsallbinary.com/java-debug-interface-api-jdi-hello-world-example-programmatic-debugging-for-beginners/
- An Intro to the Java Debug Interface (JDI): https://www.baeldung.com/java-debug-interface
- A java program to debug another java program using JDI but receive nothing useful event: https://stackoverflow.com/questions/76135113/a-java-program-to-debug-another-java-program-using-jdi-but-receive-nothing-usefu/76135372#76135372