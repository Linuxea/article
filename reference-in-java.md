# References In Java


## The problem

在所有允许动态内存分配的计算机编程语言中，有一个通用的问题，即以怎样的方式回收不再使用的内存。

这有点类似于一个餐厅，一开始你可以引导顾客到空闲的桌子。
但是不再有空桌子时，你需要检查一下之前分配好的桌子在之期间是否已经空闲下来。


有一些编程语言，如 C，将这份职责留给了开发者：你申请获得了这份内存并且现在开始由你负责来释放它。这有点像吃快餐，在你用餐完之后你需要清理你的桌子。

这当然非常高效... ...如果每个人都能做对的话。

但是如果有一些顾客忘记清理的，这很容易演变出问题。就像内存一样，很容易在某个时刻忘记释放某块区域的内存。

幸运的是，`Garbage Collectors`(下方简称 `GC`) 登上历史舞台。

> GC 最早是在 1956 年发明的，用于简化 Lisp 中的手动内存管理

有些编程语言，如 java 使用特殊的算法来收集不再使用的内存，这很棒同时对开发者非常友好。

如果内存分配与回收权利不再开发者手上了，这会引进一个新问题也是我们今天的主题：

<strong>我想保持对一个对象的引用关系，但同时不想在没有其他引用关系情况下阻止 GC 对它进行释放回收。</strong>

这有点像吃完饭在餐厅的桌子上坐了一会儿，但如果有新顾客需要餐桌，我就准备离开。


## The solution

所有开发者对 Java 中的强引用已经很熟悉了，今天就来介绍其他主角：

- `SoftReference`: 软引用
  
  由垃圾收集器根据内存需求酌情清除。经常在内存敏感的场景作为缓存使用。[...]
在虚拟机抛出 OOM 错误前，可以保证所有的软引用关联对象都被清除。

- `WeakReference`: 弱引用

    不阻止弱引用关联对象被标记为 `finalizable`, 被 `finalized` 以及回收。弱引用经常被用于映射只可达对象实例。

- `PhantomReference` 虚引用
    
    在确定虚引用关系对象被回收时，会将其进行入队操作。虚引用经常被用机制更加灵活的事后清理工作。与软引用和弱引用相同，虚引用在排队时都会被垃圾收集器自动清除。 通过虚引用可访问的对象将永远保持状态(null)，直到所有此类引用都被清除或它们自身变得不可访问为止。

所以简单来讲：软引用试图保持关联关系，弱引用不尝试保持关联关系，虚引用对象在释放后完成事后清理工作。

再次引用之前的例子：

软件引用就像一位用餐顾客，在没有其他可用的桌子后再离开餐厅。

弱引用就像只要有新顾客来就离开餐厅。

虚引用就像只要有新顾客来就离开餐厅，但是会通知餐厅经理。


## The Code

先尝试创建一个实体对象，用来作为各种关联的实体：
```java
public class Person {

  public Person(Integer id, String name, Integer age) {
    this.id = id;
    this.name = name;
    this.age = age;
  }

  Integer id;

  String name;

  Integer age;


  @Override
  public String toString() {
    return "Person{" +
        "id=" + id +
        ", name='" + name + '\'' +
        ", age=" + age +
        '}';
  }
}
```


### SoftReference


我们通过 `SoftReference` 其中的一个构造器 `SoftReference(T referent)` 创建一个软引用实体。

```java
/**
    * Creates a new soft reference that refers to the given object.  The new
    * reference is not registered with any queue.
    *
    * @param referent object the new soft reference will refer to
    */
public SoftReference(T referent) {
    super(referent);
    this.timestamp = clock;
}
```


```java
package com.linuxea.reference;

import java.lang.ref.SoftReference;
import java.util.LinkedList;
import java.util.List;

/**
 * 软引用
 */
public class SoftReferencePerson extends SoftReference<Person> {

  public SoftReferencePerson(Person referent) {
    super(referent);
  }

  public static void main(String[] args){
    SoftReferencePerson softReferencePerson = new SoftReferencePerson(new Person(0, "linuxea", 18));

    /* Force releasing SoftReferences */
    try {
      final List<long[]> list = new LinkedList<>();
      while(true) {
        list.add(new long[102400]);
      }
    }
    catch(final OutOfMemoryError e) {
      /* At this point all SoftReferences have been released - GUARANTEED. */
    }

    Person person = softReferencePerson.get();
    System.out.println(person);


  }
}
```
```console
Person{id=0, name='linuxea', age=18}
null
```

通过创建一个关联 `Person` 对象的软引用，在手动触发 `OOM` 的前后打印通过 `softReferencePerson` 获取到的结果。
可以看到 `OOM` 之前软引用的值获取时已经为 `null`。


### WeakReference

我们通过 `WeakReference` 其中一个构造器 `WeakReference(T referent)` 创建一个弱引用实体。
```java
/**
    * Creates a new weak reference that refers to the given object.  The new
    * reference is not registered with any queue.
    *
    * @param referent object the new weak reference will refer to
    */
public WeakReference(T referent) {
    super(referent);
}
```


```java
package com.linuxea.reference;

import java.lang.ref.WeakReference;

/**
 * 弱引用
 */
public class WeakReferencePerson extends WeakReference<Person> {

  public WeakReferencePerson(Person referent) {
    super(referent);
  }

  public static void main(String[] args){
    WeakReferencePerson softReferencePerson = new WeakReferencePerson(new Person(0, "linuxea", 18));
    System.out.println(softReferencePerson.get());
    System.gc();
    Person person = softReferencePerson.get();
    System.out.println(person);
  }
}
```
```console
Person{id=0, name='linuxea', age=18}
null
```

弱引用对象在下一轮的 GC 触发后将会被回收。

> Calling the gc method suggests that the Java Virtual Machine expend effort toward recycling unused objects in order to make the memory they currently occupy available for quick reuse.
> 手动调用 System.gc() 仅仅是建议虚拟机进行 GC. 如果没有触发就多试几次


### PhantomReference

`PhantomReference` 的构造器有些不同，只有一个 `PhantomReference(T referent, ReferenceQueue<? super T> q)`
```java
/**
    * Creates a new phantom reference that refers to the given object and
    * is registered with the given queue.
    *
    * <p> It is possible to create a phantom reference with a <tt>null</tt>
    * queue, but such a reference is completely useless: Its <tt>get</tt>
    * method will always return null and, since it does not have a queue, it
    * will never be enqueued.
    *
    * @param referent the object the new phantom reference will refer to
    * @param q the queue with which the reference is to be registered,
    *          or <tt>null</tt> if registration is not required
    */
public PhantomReference(T referent, ReferenceQueue<? super T> q) {
    super(referent, q);
}
```

除了关联对象 `referent`, 还有一个队列 `q`，用来注册回收引用, 实现对象回收后事后处理。


```java
package com.linuxea.reference;

import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;
import java.util.concurrent.TimeUnit;

/**
 * 虚引用
 */
public class PhantomReferencePerson extends PhantomReference<Person> implements Runnable {


  private final ReferenceQueue<Person> q;
  private final Integer personId;

  public PhantomReferencePerson(Person referent, ReferenceQueue<Person> q) {
    super(referent, q);
    this.q = q;
    this.personId = referent.id;
    new Thread(this).start(); //start a thread to listen queue
  }


  @Override
  public void run() {
    while(true) {
      try {
        // 当虚引用关系对象时回收会有通知 
        // 在 Java 9 之后，虚引用对象在入队之前就被清除了（就像其他类型的弱/软引用一样），被应用程序代码处理之前，引用对象本身已经完全死亡，这是“事后清理”
        PhantomReferencePerson reference = (PhantomReferencePerson)this.q.remove();
        // 得到通知完成一些清洁后续工作
        reference.cleanUp();
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }
    }
  }

  private void cleanUp() {
    System.out.println(this.personId + " is over");
    // do some clean up
  }



  public static void main(String[] args) throws InterruptedException {
    PhantomReferencePerson softReferencePerson = new PhantomReferencePerson(new Person(0, "linuxea", 18),
        new ReferenceQueue<>());
    // 虚引用关联对象永远不可达，因此 get 方法永远返回 null
    System.out.println(softReferencePerson.get());

    // never end
    TimeUnit.SECONDS.sleep(1000);
  }
}

```
同弱引用一样，如果未标记引用对象，则始终回收虚引用。
这一次，我们在启动 `main` 后，借用 `jvisualvm` 来触发 `GC`.

```console
null
0 is over
```
`PhantomReferencePerson` 的 `cleanUp` 被触发了。这是因为构造函数中的引用队列有了数据。

> 补充一点，软引用跟虚引用的构造函数同样拥有持有一个参数为引用队列的构造函数。 Clear phantom reference as soft and weak references do。 三者行为一致。

请记住，`object.finalize()` 方法不能保证在对象生命结束时被调用，因此如果您需要关闭文件或释放资源，您可以依赖 `PhantomReference`。 
由于 `PhantomReference` 没有指向实际对象的链接，因此典型的模式是从 `PhantomReference` 派生您自己的引用类型 `PhantomReferencePerson` 并添加一些有用的信息，例如例子中的 `personId`。






## 参考

- [1] [Weak Soft and Phantom references in Java and why they matter](https://medium.com/@ramtop/weak-soft-and-phantom-references-in-java-and-why-they-matter-c04bfc9dc792)
- [2] [Weak References in Java](https://www.baeldung.com/java-weak-reference)
- [3] [Phantom References in Java](https://www.baeldung.com/java-phantom-reference)
- [4] [Soft, weak, and phantom reference processing](https://www.ibm.com/docs/en/ztpf/2019?topic=objects-soft-weak-phantom-reference-processing)
- [5] [Clear phantom reference as soft and weak references do](https://bugs.openjdk.org/browse/JDK-8071507)
- [6] [How to make the java system release Soft References?](https://stackoverflow.com/questions/3785713/how-to-make-the-java-system-release-soft-references/3810234#3810234)
- [7] [Garbage collection (computer science)](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))


