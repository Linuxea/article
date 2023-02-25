# Lombok


程序员日常工作中经常用到的工具 `lombok` ，将众多 java 程序员从枯燥模板代码中拯救出来。
今天就来总结 `lombok` 在项目中的常用的几个注解，以此来加深印象。


## val

我们可以使用 `val` 声明局部变量的类型，而不是声明其实际类型。

- 从表达式推断出类型
- 局部变量被声明为 final

`val` 可用于局部变量与循环中，不可用于对象字段 (`field`)，同时需要声明初始表达式。


```java
package com.linuxea.lomboktutorial;

import lombok.val;
import org.junit.Test;

import java.util.ArrayList;
import java.util.function.Consumer;

public class ValTest {

    @Test
    public void testDeclareLocalVariable() {
        val name = "linuxea";
        System.out.println(name);
    }

    @Test
    public void testDeclareWithinForeach() {
        ArrayList<Integer> integers = new ArrayList<>();
        integers.add(1);
        integers.add(3);
        integers.add(5);
        integers.add(7);
        Consumer<Integer> printInteger = it -> {
            val tempInteger = it;
            System.out.println(tempInteger);
        };

        integers.forEach(printInteger);
    }

}

```


## var

`var` 跟 `val` 工作相似，除了局部变量并没有被标识为 `final`。

`var` 声明的变量类型，完全由手动初始化表达式推断出来，未来该变量的指派不再决定其另外一个合适的类型。

比如，<code>var x = "Hello"; x = Color.RED;</code> 这并不能生效。

`x` 的类型会被初始化表达式推断为 `java.lang.String`，因此 `x = Color.RED` 指派将会失败。

如果 `x` 被推断为 `java.lang.Object`，这段代码将会被编译通过，但是这并不是 `var` 的工作方式。

```java
package com.linuxea.lomboktutorial;

import org.junit.Test;

public class VarTest {

    @Test
    public void testVar() {
        var a = "string"; // var infer a as a string from initial expression
        System.out.println(a);
    }

    @Test
    public void testVarInfer() {
        var content = "this is a content of string type";

        content = 1; // compiled fail
    }

}
```



## @NonNull

你可以在字段，方法或构造器参数使用 `@NonNull` 注解。

这会使 `lombok` 生成 `null` 检查语句 (`null-check`)。
```java
package com.linuxea.lomboktutorial;

import lombok.NonNull;

public class NonnullTest {

    // Nonnull constructor parameter
    public NonnullTest(@NonNull String string) {
        // pass
    }


    // Nonnull method parameter
    public void say(@NonNull String something) {
        System.out.println(something);
    }


}

```

生成：
```java
package com.linuxea.lomboktutorial;

import lombok.NonNull;

public class NonnullTest {
    public NonnullTest(@NonNull String string) {
        if (string == null) {
            throw new NullPointerException("string is marked non-null but is null");
        }
    }

    public void say(@NonNull String something) {
        if (something == null) {
            throw new NullPointerException("something is marked non-null but is null");
        } else {
            System.out.println(something);
        }
    }
}

```

`Lombok` 在生成完整的方法或者构造器的时候，通常会把 `@Nonnull` 注解的字段作为生成非空检查的信号。
两者搭配一起工作，比如 `@Data` 注解。

生成前：
```java
package com.linuxea.lomboktutorial;

import lombok.Data;
import lombok.NonNull;

@Data
public class NonnullDataTest {

    @NonNull
    private final String content;
    
}

```

生成：
```java
package com.linuxea.lomboktutorial;

import lombok.NonNull;

public class NonnullDataTest {
    private final @NonNull String content;

    public NonnullDataTest(@NonNull String content) {
        if (content == null) {
            throw new NullPointerException("content is marked non-null but is null");
        } else {
            this.content = content;
        }
    }

    //省略 @Data 生成的 equals hashCode toString 等方法
}
```

- 普通方法生成 null-check 代码会被插入到方法内部的最开始位置
- 构造器方法生成 null-check 代码会在 this() 或者 super() 紧随其后
- 相对的参数的构造方法已经存在时，则不会生成对应的 null-check 代码



## @Cleanup

`@Cleanup` 来确保给定的资源在退出代码的可执行路径时被自动清理。

你可以使用 `@Cleanup` 注解声明任何局部变量。
```java
@Cleanup InputStream in = new FileInputStream("some/file");
```

因此， `in` 在进入的范围最后，会自动调用 `in.close()`。
`in.close()` 的调用保证是通过 `try/finally` 结构来实现的。


如果 `@Cleanup` 注解使用的需要清理的资源并没有 `close()` 方法，而是另外的无参方法，你可以通过指定方法的名称来实现。
```java
@Cleanup("shutdown") ExecutorService executorService = Executors.newFixedThreadPool(1);
```

默认情况下，清理方法假定为 `close()`。清理方法如果有1个或多个参数，则不能通过 `@Cleanup` 注解来调用。


## @Getter and @Setter

你可以通过 `@Getter`/`@Setter` 注解的使用，来让 lombok 自动生成默认的 `getter`/`setter` 方法。

默认的 `getter` 方法简单地返回了字段值，如字段是 `foo` 则 `getter` 方法为 `getFoo`（或者字段类型是布尔时的 isFoo）。
默认的 `setter` 方法返回 `void`，如字段是 `foo` 则 `setter` 方法为 `setFoo`，并且使用了一个与字段相同类型的参数，`setter` 简单地把字段值设置为该参数对应的值。


生成的 `getter`/`setter` 默认方法级别为 `public` ，除非你通过 `AccessLevel` 属性进行显式指定。
合法的访问级别包含 `PUBLIC`, `PROTECTED`, `PACKAGE` 与 `PRIVATE`。


你也可以在把 `@Getter` 与 `@Setter` 放在类上。
在这种情况下，就好像你用注释注释了那个类中的所有非静态字段。

你可以通过指定特殊的访问级别为 `AccessLevel.NONE` ，来禁止任何字段生成 `getter`/`setter`。
这种方式可以覆盖类上的 `@Getter`，`@Setter` 或者 `@Data` 注解在该字段上产生的行为。


为了在生成的方法上面放置需要的注解，你可以使用属性 `onMethod=@__({@AnnotationsHere})`; 
如果想要在 `setter` 方法的唯一参数放置注解，可以使用属性 `onParam=@__({@AnnotationsHere})`。



## @ToString

使用 `@ToString` 来注解一个类，会让 `lombok` 生成 `toString()` 方法的实现。

你可以使用配置选项来指定是否哪些字段需要被包含，否则其格式是固定的：类名后跟括号，其中包含以逗号分隔的字段，例如 我的类（foo=123，bar=234）。


通过设置 `includeFieldNames` 参数为 `true`，输出字段的名称而不仅仅是值，这可以增加 `toString()` 方法的输出清晰度（同时对应长度也会增加）。

默认情况下，`non-static` 字段会被打印。如果你想跳过某些字段，你可以通过注解 `@ToString.Exclude` 标识这些字段。

除此之外，你也可以通过 `@ToString(onlyExplicitlyIncluded = true)` 以及 `@ToString.Include` 明确标识每个需要包含的字段。



通过设置 `callSuper` 为 `true`, 你可以包含父类的 `toString` 的实现。

需要注意的是，`java.lang.Object` 中 `toString()` 方法的实现是没有太多意义的，所以你可能不太想这样做，除非你继承了其他的类。

你也可以在 `toString()` 的输出包含一个方法调用的输出。
只有无参的实例方法才能够被包含。

你可以通过将方法标识为 `@ToString.Include` 来实现这一点。


你可以通过 `@ToString.Include(name = "another name")` 来改变 name 以此来标识成员， `@ToString.Include(rank = -1)` 来改变成员打印的顺序。

没有标识 `rank` 属性的成员会被认为拥有值为 `0` 的排序，较高的排序会被优先打印，相同的排序成员会按照源码中出现的顺序打印。


```java
@ToString(onlyExplicitlyIncluded = true)
public class ToStringTest {

    @ToString.Include(name = "anotherName", rank = 2)
    private String name;

    private Integer age;
}
```


## @EqualsAndHashCode

任何类定义都可以用 `@EqualsAndHashCode` 注释，让 `lombok` 生成 `equals(Object other)` 和 `hashCode()` 方法的实现。

默认情况下，它会使用所有 `non-static`、`non-transient` 字段，不过你可以通过 `@EqualsAndHashCode.Include` or `@EqualsAndHashCode.Exclude` 标识字段来更改哪些字段会被使用，甚至指定使用多个方法的输出。

```java
package com.linuxea.lomboktutorial;

import lombok.EqualsAndHashCode;

@EqualsAndHashCode
public class EqHashTest {

    @EqualsAndHashCode.Include
    private String parent;

    @EqualsAndHashCode.Exclude
    private String son;
    
}

```


## @NoArgsConstructor, @RequiredArgsConstructor, @AllArgsConstructor

这组三个注解会生成各自对应的某个字段对应的一个参数，并且简单地将参数赋值给该字段。


`@NoArgsConstructor` 会生成一个不带任何参数的构造器。
如果不可能出现这种情况（比如只存在 `final` 字段)，则会出现编译异常，除非使用 `@NoArgsConstructor(force = true)`，
所有的 `final` 字段将会被初始化为 `0`/`false`/`null`。
同时对于有约束的字段，比如 `@NonNull` 字段，不会有相应的 `null-check` 代码生成。

`@RequiredArgsConstructor` 为每个需要特殊处理的字段生成一个带有 1 个参数的构造函数。 
- 所有未初始化的 final 字段
- 标记为 @NonNull 且未在声明处初始化的任何字段

对于那些标有 @NonNull 的字段，还会生成显式空检查。 
如果用于标有 @NonNull 的字段的任何参数包含 null，则构造函数将抛出 NullPointerException。 
参数的顺序与字段在您的类中出现的顺序相匹配。


`@AllArgsConstructor` 为类中的每个字段生成一个带有 1 个参数的构造函数。 标有 @NonNull 的字段会导致对这些参数进行空检查。

以上三个注解同时允许有另外一种形式：其中生成私有的构造函数，与额外的一个包围私有构造方法的静态工厂方法。

这可以通过为注解提供 `staticName` 属性一个值来实现。
比如：
```java
package com.linuxea.lomboktutorial;


import lombok.AllArgsConstructor;
import lombok.RequiredArgsConstructor;

@AllArgsConstructor(staticName = "of")
@RequiredArgsConstructor(staticName = "of")
public class StaticFactoryMethodArgsTest {

    private final Integer age;
    private String name;
}
```


为了在生成的构造器上放置注解，可以使用 `onConstructor=@__({@AnnotationsHere})`。


## @Data


`@ToString`、`@EqualsAndHashCode`、所有字段上的 `@Getter`、所有非最终字段上的 `@Setter` 和`@RequiredArgsConstructor` 组合成的快捷方式



## @Value


`@Value` 是 `@Data` 不可变形态的变体。

所有的字段默认都被设置为 `private` 与 `final`，并且不会生成 `setter` 方法。

默认情况下，类本身也是最终的，因为不变性不是可以强加给子类的东西。

与 `@Data` 相似，有用的方法如 `toString()`, `equals()` 与 `hashCode()` 都会被生成，所有的字段都会有一个 `getter` 方法，并且会有生成一个覆盖每个参数（除了声明时初始化的字段）的构造器。

```java
package com.linuxea.lomboktutorial;

import lombok.Value;

@Value
public class ValueTest {

    String name;

    Integer age;

    String sex = "boy";

}
```

## @Builder

@Builder 为我们生成了建造者模式代码。
它可以用在类，构造器和方法上。

```java
package com.linuxea.lomboktutorial;


import lombok.Builder;

@Builder(builderClassName = "classBuilder")
public class BuilderTest {


    @Builder.Default
    private String name = "linuxea";
    @Builder.Default
    private Integer age = 20;


    @Builder(builderClassName = "constructorBuilder")
    public BuilderTest(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    @Builder(builderClassName = "methodBuilder")
    public BuilderTest make(String name, Integer age) {
        return new BuilderTest(name, age);
    }

    void test() {
        // class builder
        BuilderTest classBuilder = new classBuilder().name("linuxea").age(18).build();
        System.out.println(classBuilder);

        // constructor builder
        BuilderTest constructorBuilder = new constructorBuilder().name("linuxea").age(18).build();
        System.out.println(constructorBuilder);

        // method builder
        BuilderTest methodBuilder = new methodBuilder().name("linuxea").age(18).build();
        System.out.println(methodBuilder);
    }
}

```

## SneakyThrows

`@SneakyThrows` 可用于偷偷抛出受检异常，而无需在方法的 throws 子句中实际声明它。

躲避受检异常捕获机制的使用场景，通常有两种情况：
- 非必要：异常的传播无论是否捕获，都原封交由外部处理
```java
    @SneakyThrows
    @Override
    public void run() {
        throw new IOException();
    }
```


- 不可能的异常：例如，<code>new String(someByteArray, "UTF-8");</code> 声明中它可以抛出 `UnsupportedEncodingException`， 但根据 JVM 规范，UTF-8 必须始终可用
```java
    @SneakyThrows
    public String print() {
        return new String("abc".getBytes(), StandardCharsets.UTF_8);
    }
```



## @Synchronized

`@Synchronized` 是 `synchronized` 方法修饰法的变体。

与 `Synchronized` 相似，这个注解也只能被用到静态与实例方法。它的操作也类似于 `synchronized`，但是它锁定不同的对象。

关键词 `synchronized` 锁定了 `this` 当前对象，但是 `@Synchronized` 锁定一个私有的 `$lock` 字段。
如果该字段不存在，则会自动创建生成。

如果你注解一个静态方法，`@Synchronized` 锁定一个命名为 `$LOCK` 的静态字段。
```java
package com.linuxea.lomboktutorial;

import lombok.Synchronized;

public class SynchronizedTest {


    @Synchronized
    public void testMethod() {
    }

    @Synchronized
    public static void testMethod2() {

    }

}
```

## @With

为一个不可变属性设值，下一个最好的方法是构造一个对象的克隆，但是为这个字段提供一个新的值。生成此克隆的方法正是 `@With` 生成的方法：一个 `withFieldName(newValue)` 方法，它生成除了关联字段的新值之外的克隆。

比如，如果你创建一个类 <code>public class Point { private final int x, y; }</code> ，因为所有的字段都是 `final`, 所以 `setter` 方法是无意义。 `@With` 注解能够生成 withXXX(int newValue) 方法，使用提供的新 x 值与相同的旧值 y 来返回一个新的 Point 对象。

`@With` 依赖一个所有字段的的构造器来完成工作。如果这个构造器不存在，则会导致编译异常。

你可以使用 lombok 的 @AllArgsConstructor 或者 @Value 来自动生成一个所有参数的构造器。
当然同样可接受的是你手工创建这个构造器，它必须包含所有以相同的词汇顺序的 non-static 字段。
```java
package com.linuxea.lomboktutorial;

import lombok.With;

public class WithTest {

    @With
    private final String name;

    @With
    private final Integer age;

    private Integer abe;


    public WithTest(String name, Integer age) {
        this.name = name;
        this.age = age;
    }


    public WithTest(String name, Integer age, Integer abe) {
        this.name = name;
        this.age = age;
        this.abe = abe;
    }
}

```

## @Getter(lazy=true)

可以让 lombok 生成一个 getter，它会在第一次调用此 getter 时计算一次值，并缓存它。 如果计算该值需要大量 CPU，或者该值需要大量内存，这将很有用。 

要使用此功能，请创建一个 `private` `final` 变量，使用运行成本高的表达式对其进行初始化，
并使用 `@Getter(lazy=true)` 注释该字段。 在首次调用该字段的 `getter` 时，表达式将被计算不超过一次。 

```java
package com.linuxea.lomboktutorial;

import lombok.Getter;
import org.junit.Test;

public class LazyTest {

    @Getter(lazy = true)
    private final String expensive = expensiveCalculate();

    public String expensiveCalculate() {
        System.out.println("expensiveCalculate");
        return "abc";
    }

    @Test
    public void testLazy() {
        LazyTest lazyTest = new LazyTest();
        System.out.println(lazyTest.getExpensive());
        System.out.println(lazyTest.getExpensive());
        System.out.println(lazyTest.getExpensive());
    }

}
```



# Summary

简要了解了常用的几个 lombok 注解，这对于日常开发工作能够提高一定的效率。
对于 java 这门古老且经常存在样板化代码的语言，带来了一定的新改变。

同时，lombok 提供了更多实验性质的注解，这将后面的学习中会更加深入去了解。



# Reference

- [1] [projectlombok](https://projectlombok.org/features/)
- [2] [project lombok tutorial](https://github.com/Linuxea/lomboktutorial)