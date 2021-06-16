# Chap2

## Immutable pattern

### 介绍

Immutable 就是不变的，不发生改变的意思。因为实例内部状态不会变化，所以实例不论被多少个线程访问，也无需进行互斥的处理。

### 特性

多线程能自由访问具有 `Immutable` 角色的对象，而无需使用 `Single Threaded Execution` 加以限定。

```java
public final class Person {
    private final String name;
    private final String address;

    public Person(String name, String addr) {
        this.name = name;
        this.address = addr;
    }
    //...
}
```

### 拓展

* 何时使用

  1）实例创建后，状态不再发生变化

  字段申明为`final`，且无对应的`setter`方法。注意，即使字段的值不会发生变化，字段引用的实例也有可能会变 (e.g. 字段为 `StringBuffer` 类型)。

  2）实例是共享的，且被频繁访问

  其优点是不需要使用 `synchronized` 进行保护，即意味着能够在不失去安全性和活性的前提下提升性能。当实例被多个线程共享，且有可能被频繁访问时，模式的优点就会体现出来。

* 考虑成对的mutable类与immutable类 [性能]

  例如针对字符串的概念，Java标准类库中的mutable类和immutable类: String 与StringBuffer。

  String 与StringBuffer可以相互转化，如果需要频繁修改字符串内容，使用StringBuffer；如果不修改字符创的内容，只是引用其内容，则使用String。

### 延伸

集合类与多线程

* 非线程安全的 `java.util.ArrayList` 类

* 利用 `Collections.synchronizedList` 方法进行同步

* 使用写时复制（`copy-on-write`）的 `java.util.concurrent.CopyOnWriteArrayList` 类

  使用写时复制 copy-on-write 时，每次写入都会执行复制，因而如果频繁地执行“写”操作，则会比较耗时；而如果写操作非常少，读操作频率非常高时就极为适用了。
