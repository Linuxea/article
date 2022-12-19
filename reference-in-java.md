# References In Java


## The problem
One common problem of all computer languages that allow for dynamic memory allocation, is to find a way to “collect” the memory after is not used anymore.

在所有允许动态内存分配的计算机编程语言中，有一个通用的问题，即以怎样的方式回收不再使用的内存。

It is a bit like in a restaurant, at the beginning you can accommodate customers to the empty tables, but when you don’t have empty tables anymore, you need to check if some of the tables already allocated have got free in the meanwhile.


这有点类似于一个餐厅，一开始你可以引导顾客到空闲的桌子，但是不再有空桌子时，你需要检查一下之前分配好的桌子在此期间是否已经空闲下来。

Some languages, like C, leave the responsibility to users: you have got the memory and now it is your responsibility to free it. It’s a bit like in a fast food, where you are supposed to clean up your table after the meal.

有一些编程语言，如 C，将这份职责留给了开发者：你获得了这份内存并且现在开始由你负责来释放它。这有点像吃快餐，在你用餐完之后你需要清理你的桌子。


This is very efficient… if everybody behaves correctly. But if some customers forget to clean up, it will easily become a problem. Same with memory: it’s very easy to forget to free an area of memory.

这当然非常高效... ...如果每个人都能做对的话。

但是如果有一些顾客忘记清理的，这很容易演变出问题。就像内存一样，很容易在某个时刻忘记释放某块区域的内存。

So Garbage Collectors (GC from here on) come to help. Some languages, namely Java, use a special algorithm to collect all the memory which is not used. Which is very nice of them and very convenient for programmers. You may be forgiven if you think that GC is a relatively recent technique.

幸运的是，Garbage Collectors(下方简称 GC) 登上历史舞台。

有些编程语言，如 java 使用特殊的算法来收集不再使用的内存，这很棒同时对开发者非常友好。


This anyway introduce a new problem: what if we want to keep a reference to an object but we don’t want to prevent GC to free it if there is no other reference? It’s a bit like sitting a while on a table at restaurant after having finished but be ready to leave if a new customer needs the table.

如果内存分配与回收权利不再开发者手上了，这会引进一个新问题：

我想保持对一个对象的引用关系，但同时不想在没有其他引用关系情况下阻止 GC 对它进行释放回收。

这有点像吃完饭在餐厅的桌子上坐了一会儿，但如果有新顾客需要餐桌，我就准备离开。


## The solution

所有开发者对 Java 中的强引用已经很熟悉了，今天就来介绍其他主角：

SoftReference Soft reference objects, which are cleared at the discretion of the garbage collector in response to memory demand. Soft references are most often used to implement memory-sensitive caches. [..]
All soft references to softly-reachable objects are guaranteed to have been cleared before the virtual machine throws an OutOfMemoryError.


SoftReference: 软引用，由垃圾收集器根据内存需求酌情清除。经常在内存敏感的场景作为缓存使用。[...]
在虚拟机抛出 OOM 错误前，可以保证所有的软引用关联对象都被清除。


WeakReference Weak reference objects, which do not prevent their referents from being made finalizable, finalized, and then reclaimed. Weak references are most often used to implement canonicalizing mappings. [Here Canonicalizing mappings means mapping only reachable object instances.]

WeakReference: 弱引用，不阻止弱引用关联对象被标记为 finalizable, 被 finalized 以及回收。弱引用经常被用于映射只可达对象实例。


PhantomReference Phantom reference objects, which are enqueued after the collector determines that their referents may otherwise be reclaimed. Phantom references are most often used for scheduling pre-mortem cleanup actions in a more flexible way than is possible with the Java finalization mechanism.[..]
Unlike soft and weak references, phantom references are not automatically cleared by the garbage collector as they are enqueued. An object that is reachable via phantom references will remain so until all such references are cleared or themselves become unreachable.

PhantomReference 虚引用，在确实虚引用关系对象可以被回收时，会将其进行入队操作。虚引用经常被用在比 Java finalization 机制更加灵活的事前清理工作。
与软引用和弱引用不同，虚引用在排队时不会被垃圾收集器自动清除。 通过虚引用可访问的对象将保持此状态(null)，直到所有此类引用都被清除或它们自身变得不可访问为止。

So in brief: Soft references try to keep the reference. Weak references don’t try to keep the reference. Phantom references don’t free the reference until cleared.

所以简单来讲：软引用试图保持关联关系，弱引用不尝试保持关联关系，虚引用在完成事前清理工作前不会被释放。

再次引用之前的例子：
a SoftReference is like a customer that say: I’ll leave my table only when there are no other tables avalaible. A WeakReference is like someone ready to leave as soon as a new customer arrives. A PhantomReference is like someone ready to leave as soon as a new customer arrives, but actually not leaving until the manager gives him permission.

软件引用就像一位用餐顾客，在没有其他可用的桌子后再离开餐厅。
弱引用就像只要有新顾客来就离开餐厅。
虚引用就像只要有新顾客来就离开餐厅，但前提是必须得到餐厅经理的同意。



## The Code

