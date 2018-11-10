---
title: 理解 Java 中的 volatile
date: 2018-11-10 12:39:10
author: zjmeow
tags:
	- Java
	- 16级
---

标题 neta 自《计算机网络自顶向下》

思维导图



![](https://img2018.cnblogs.com/blog/1215522/201810/1215522-20181014194730074-1616119440.png)




volatile 在 Java 中被称为轻量级 `synchronized`。很多并发专家引导用户远离 volatile 变量，因为使用它们要比使用锁更加容易出错。但是如果理解了 volatile 能帮助你写出更好的程序。

* 当读比写更多时会获得比锁好相当多的性能
* 比锁更好的伸缩性
* 比锁使用方便，只需要声明变量即可，代码量小







# 内存语义

### volatile 的讲解

为了方便理解 volatile，用代码来表示一下加了 volatile 的效果。

给变量加上 volatile 相当于在 get 和 set 方法中加了锁。

```java
public synchronized int getX() {
    return x;
}

public synchronized void setX(int x) {
    this.x = x;
}
```

注意这里只保证了get 和 set 的原子性，当有其他操作的时候就不是原子性的了。



下面的操作不是原子性的，当个 5 个线程同时执行这个方法 100 次后出现的结果很可能小于 500。

```java
volatile int x;
public void inc() {
    x++;
}
```

原因是这个程序相当于

```java
int x;
public synchronized int getX() {
    return x;
}

public synchronized void setX(int x) {
    this.x = x;
}
public void inc() {
    int temp = getX(); // 1
    temp += 1; // 2
    setX(temp); // 3
}
```



可以看出即使 get 和 set 操作是原子性的，整个操作也不是原子性的。

当两个线程 A ， B 同时执行  `inc` 时，可能会出现 

A-1 得到 x = 1 

B-1 得到 x = 1

A-2 temp 为 2

B-2 temp 为 2

A-3 x 被设为 2

B-3 x 被设为 2



在执行完毕后 x 的值只增加了 1。





### 原子性和可见性

我们在 JMM 中讲解 volatile 的内存语义。可以参照着这篇看。[JVM内存模型、指令重排、内存屏障概念解析](https://www.cnblogs.com/chenyangyao/p/5269622.html)

volatile 保证了新的值能立刻同步到主内存中，以及每次使用前都到主内存刷新。

![](https://img2018.cnblogs.com/blog/1215522/201810/1215522-20181014194746418-166343032.png)


volatile 通过在写入变量的时候，JVM 会向 CPU 发送一个 lock 前缀指令将变量同步入主内存

而当出现了这个命令以后，所有其他线程上的缓存就会被强制设置成无效，当下次要用到这个变量的时候需要去主内存中取。

通过 Lock 指令

每次使用变量之前都必须先从主内存刷新最新的值。

每次修改变量后都必须立即同步回主内存中，保证其他线程可以看到最新的值。



**一个比较有用的抽象：把加了 volatile 的变量当作是没有中间的缓存，所有的数据操作都是在主内存上的。**



###禁止指令重排

CPU 和编译器为了执行效率，会将指令重排序。如果不知道的可以参照上面那一片博文来对照着读。

volatile 修饰变量不会被指令重排优化，保证代码执行顺序和程序顺序相同。

在几个地方会插入 **StoreStore** 和 **StoreLoad** 阻止重排序。

* 在 volatile 写前插入 StoreStore 屏障，写后插入 StoreLoad 屏障
* 在 volatile 读前插入 LoadLoad 屏障，读后插入 LoadStore 屏障



如果不知道这两个指令可以看一下上面的博客。 // TODO 马上写完 (咕咕咕











# 正确的使用 volatile

正确使用 volatile 依赖于

- 对变量的写操作不依赖于当前值
- 该变量没有包含在具有其他变量的不变式中


### 模式 #1：状态标志

也许实现 volatile 变量的规范使用仅仅是使用一个布尔状态标志，用于指示发生了一个重要的一次性事件，比如游戏结束，将游戏正在进行的 flag 设置为 false，通知绘图线程停止。

```java
volatile boolean flag = false;
private void waiting() {
    while(!flag) {
        // do something
    }
}
```





### 模式 #2：一次性安全发布

一次性安全分布用于双重检查实现单例模式。

```java
private volatile SingleTest instance;
SingleTest getInstance() {
    if (instance == null) {
        synchronized (SingleTest.class) {
            if (instance == null) {
                instance = new SingleTest();
            }
        }
    }
    return instance;
}
```

为什么要用到 volatile 呢？因为新建类分为三步

* 分配内存空间
* 初始化对象
* 设置内存地址，初始化引用

在这里第二步可能重排序，这时候可能会将没有初始化成功就把对象发布出去了，所以需要 volatile 来阻止指令重排。



### 模式 #3：独立观察

安全使用 volatile 的另一种简单模式是：定期 “发布” 观察结果供程序内部使用。例如，假设有一种环境传感器能够感觉环境温度。一个后台线程可能会每隔几秒读取一次该传感器，并更新包含当前文档的 volatile 变量。然后，其他线程可以读取这个变量，从而随时能够看到最新的温度值。

使用该模式的另一种应用程序就是收集程序的统计信息。清单 4 展示了身份验证机制如何记忆最近一次登录的用户的名字。将反复使用 `lastUser` 引用来发布值，以供程序的其他部分使用。

该模式是前面模式的扩展；将某个值发布以在程序内的其他地方使用，但是与一次性事件的发布不同，这是一系列独立事件。这个模式要求被发布的值是有效不可变的 —— 即值的状态在发布后不会更改。使用该值的代码需要清楚该值可能随时发生变化。

### 模式 #4：volatile bean 模式

volatile bean 是线程安全的。在 volatile bean 模式中，JavaBean 被用作一组具有 getter 和/或 setter 方法 的独立属性的容器。

在 volatile bean 模式中，

* JavaBean 的所有数据成员都是 volatile 类型的
* 并且 getter 和 setter 方法除了获取或设置相应的属性外，不能包含任何逻辑。、
* 此外，对于对象引用的数据成员，引用的对象必须是有效不可变的。（这将禁止具有数组值的属性，因为当数组引用被声明为 `volatile` 时，只有引用而不是数组本身具有 volatile 语义）



```java
@ThreadSafe
public class Person {
    private volatile String firstName;
    private volatile String lastName;
    private volatile int age;
 
    public String getFirstName() { return firstName; }
    public String getLastName() { return lastName; }
    public int getAge() { return age; }
 
    public void setFirstName(String firstName) { 
        this.firstName = firstName;
    }
 
    public void setLastName(String lastName) { 
        this.lastName = lastName;
    }
 
    public void setAge(int age) { 
        this.age = age;
    }
}
```








# 对于 volatile 的优化

在 JDK 7 并发包里新增了一个队列集合 LinkedTransferQueue，它在使用 volatile 变量时，用一种追加字节的方式来优化出队和入队的性能。

它将变量追加到了 64 字节来提高性能。

64 位 CPU 在队列中的元素不足 64 个字节时会将多个元素读入一个缓存行中，在多线程当读取一个元素的时候会锁住这个缓存行，进而导致这个元素附近的元素都不能被读取。 

如果一个变量为 64 字节，那么每个元素都被读入不同的缓存中，相邻队列元素就能被不同线程同时访问了。



参考文献

* 周志明. 深入理解 Java 虚拟机 [M]. 机械工业出版社, 2011.
* 方腾飞.Java 并发编程的艺术 [M]. 机械工业出版社, 2015.
* [正确使用 Volatile 变量](https://www.ibm.com/developerworks/cn/java/j-jtp06197.html)
* [JVM内存模型、指令重排、内存屏障概念解析](https://www.cnblogs.com/chenyangyao/p/5269622.html)