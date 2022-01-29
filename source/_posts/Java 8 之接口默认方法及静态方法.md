---
title: Java 8 之接口默认方法及静态方法
toc: true
date: 2019-07-11 17:44:15
tags: Java 8
categories: Java 基础
---

## 概要

Java 8 新增了很多语言特性，其中涉及接口的有**默认方法**、**静态方法**，这些语言特性旨在帮助编写类库的开发人员，但对我们日常写应用程序的开发人员也同样适用。

静态方法旨在替代常见的伴生类，例如 Collection 类的伴生类 Collections 类、Path 类的半生类 Paths。

默认方法一来解决接口演化问题，二来替代抽象骨干类，例如 Collection 接口的抽象骨干类 AbstractCollection 类。



## 默认方法

默认方法，简而言之，就是在一个接口里使用 `default ` 关键字定义的一个方法，所有该接口的实现类都可以直接使用而无须自己实现此方法。

默认方法存在的主要目的是为了方便扩展已有接口，如果没有默认方法，给一个已经发布的接口添加方法时，所有实现改接口的地方都得做修改，如果该接口的使用范围很广，那相应的影响也非常之大，例如 JDK 里定义的各种接口。因此，Java 8 中加入了此语言特性使得类库设计人员可以很方便的扩展已有的接口。



### 重写规则

实现包含默认方法的接口，有以下三种可能的方式：

- 实现类不重写默认方法，直接使用接口中的默认方法；
- 子接口或抽象类将默认方法声明为普通的抽象方法；
- 实现类中重写默认方法，与重写普通的接口方法一样。

如果使用代码片段描述这三种方式则如下所示：

```java
public interface Parent {
    void message(String body);

    default void welcome(){
        message("Parent: Hi!");
    }
}

// 1. 不对默认方法做任何重写，直接使用
public class Child implements Parent {
    public void message(String body) {
        System.out.println(body);
    }
}

// 2. 子接口或抽象类重新声明默认方法
public abstract BaseParent implements Parent {
    abstract void welcome();
}

// 3. 实现类重写默认方法
public class Child implements Parent {
    public void message(String body) {
        System.out.println(body);
    }

    public void welcome() {
        message("Child: Hi!");
    }
}
```

如果一个复杂的继承体系里，存在多个子类或子接口都重写了默认方法，到底哪个其作用呢？可以使用一句话简单概括其规则：**类中重写的方法胜出。**这样的设计主要是由增加默认方法的目的决定的，增加默认方法主要是为了在接口上向后兼容。让类中重写方法的优先级高于默认方法能简化很多继承问题



### 多重继承问题

接口允许多重继承，因此有可能碰到两个接口包含签名相同的默认方法的情况，如果一个类同时实现这两个接口，它到底该继承哪个接口的呢？

```java
public interface Foo {
    default String hello() {
        return "hello, Foo!"
    }
}

public interface Bar {
    default String hello() {
        return "hello, Bar!";
    }
}

public class FooBar implements Foo,Bar {
    // 编译失败，编译器提示：class FooBar inherits unrelated defaults for hello() from types Foo and Bar.
}
```

解决此问题，我们需要显示的实现或声明 `hello()` 方法，如下所示：

```java
public class FooBar implements Foo,Bar {
    public String hello() {
        // 实现逻辑
        return Foo.super.hello();
    }
}

public abstract class BaseFooBar implements Foor, Bar {
    public abstract String hello();
}
```



## 静态方法

我们在编程过程中，有时会将创建一些工具类，这些类里包含了大量的静态方法，例如 Objects 类、Collections 类等，这些工具方法，在语义上它们不具体属于某个类。但如果一个方法有充分的语义原因和某个概念相关，那么就应该将该方法和相关的类或接口放在一起，而不是放到另一个工具类中。

例如，Java 8 中引入的 `Stream` 接口，如果想创建一个由简单值组成的`Stream`，自然希望`Stream`接口就有一个这样的方法，而不是使用额外的工具类。这在以前很难达成，直到 Java 8 中为接口加入了静态方法，如下所示：

```java
public static<T> Stream<T> of(T... values) {
    return Arrays.stream(values);
}
```



## 参考资料

[The Java™ Tutorials - Default Methods](https://docs.oracle.com/javase/tutorial/java/IandI/defaultmethods.html)

[Java 8函数式编程](http://www.ituring.com.cn/book/1448)
