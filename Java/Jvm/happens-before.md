# JMM 的设计

每个 Java 线程在执行代码的过程中，都会把主内存中的共享变量 load 一份副本到工作内存中。

![](https://mmbiz.qpic.cn/mmbiz_jpg/A3ibcic1Xe0iaQ3bQGCubgA1AePT7Iq1t9rwpPpdREaZePsUj7KZX11miaibVtXA42dGA4G2U5G9CwCIPsB0027NYdQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

当每个 Java 线程修改工作内存中的共享变量副本后，会再把共享变量 store 到主存中，由于不同线程对共享变量的修改不一样，而且每个线程对共享变量的修改彼此不可见，所以最后覆盖内存中共享变量的值的时候可能会出现重复覆盖的现象，这也是共享变量不安全的因素。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/A3ibcic1Xe0iaQ3bQGCubgA1AePT7Iq1t9rf8SnjJb7CIx5EMicngqFcEAcEuRv1LHrcurQMSibn2sII3CN6SowdNZQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

由于 JMM 的这种设计，导致出现了我们经常说的**可见性**和**有序性**问题。

# happens - before 原则

JSR-133 使用 happens - before 原则来指定两个操作之间的执行顺序。这两个操作可以在同一个线程内，也可以在不同线程之间。同一个线程内是可以使用 `as-if-serial` 语义来保证可见性的，所以 happens - before 原则更多的是用来解决不同线程之间的可见性。

## 程序顺序规则

Each action in a thread happens-before every subsequent action in that thread.

每个线程在执行指令的过程中都相当于是一条顺序执行流程：取指令，执行，指向下一条指令，取指令，执行。

![图片](https://mmbiz.qpic.cn/mmbiz_png/A3ibcic1Xe0iaQ3bQGCubgA1AePT7Iq1t9rtyeJPVwwRl4kerm4pNib11zg10Bv92DVZxr3DRGNaIwiayWhZAAdbR1g/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

而程序顺序规则说的就是在同一个顺序执行流中，会按照程序代码的编写顺序执行代码，编写在前面的代码操作要 happens - before 编写在后面的代码操作。

> 这里需要特别注意⚠️的一点就是：这些操作的顺序都是对于同一个线程来说的。

## monitor 规则

An unlock on a monitor happens-before every subsequent lock on that monitor.

这是一条对 monitor 监视器的规则，主要是面向 lock 和 unlock 也就是加锁和解锁来说明的。这条规则是对于同一个 monitor 来说，这个 monitor 的解锁（unlock）要 happens - before 后面对这个监视器的加锁（lock）。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/A3ibcic1Xe0iaQ3bQGCubgA1AePT7Iq1t9rN8R23s9sibuiaBqwHmdVaKvpROQZ1az4uMNEEOKKlCu9SwiafZfyCI6vQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

比如下面这段代码

```java
class monitorLock {
    private int value = 0;
  
    public synchronized int getValue() {
        return value;
    }
    
    public synchronized void setValue(int value) {
        this.value = value;
    }
}
```

在这段代码中，getValue 和 setValue 这两个方法使用了同一个 monitor 锁，假设 A 线程正在执行 getValue 方法，B 线程正在执行 setValue 方法。monitor 的原则会规定线程 B 对 value 值的修改，能够直接对线程 A 可见。如果 getValue 和 setValue 没有 synchronized 关键字进行修饰的话，则不能保证线程 B 对 value 值的修改，能够对线程 A 可见。

> monitor 的规则对于 synchronized 语义和 ReentrantLock 中的 lock 和 unlock 的语义是一样的。

## volatile 规则

A write to a volatile ﬁeld happens-before every subsequent read of that volatile.

这是一条对 volatile 的规则，它说的是对一个 volatile 变量的写操作 happens - before 后续任意对这个变量的读操作。

这条规则其实就是在说 volatile 语义的规则，因为对 volatile 的写和读之间会增加 memory barrier ，也就是内存屏障。

内存屏障也叫做`栅栏`，它是一种底层原语。它使得 CPU 或编译器在对内存进行操作的时候, 要严格按照一定的顺序来执行, 也就是说在 memory barrier 之前的指令和 memory barrier 之后的指令不会由于系统优化等原因而导致乱序。

## 线程 start 规则

A call to start() on a thread happens-before any actions in the started thread.

这条规则也是适用于同一个线程，对于相同线程来说，调用线程 start 方法之前的操作都 happens - before start 方法之后的任意操作。

> 这条原则也可以这样去理解：调用 start 方法时，会将 start 方法之前所有操作的结果同步到主内存中，新线程创建好后，需要从主内存获取数据。这样在 start 方法调用之前的所有操作结果对于新创建的线程都是可见的。

![图片](https://mmbiz.qpic.cn/mmbiz_jpg/A3ibcic1Xe0iaQ3bQGCubgA1AePT7Iq1t9rcjGMNO1IC1I6ueMIBzUqZuOC0uibyKb6jiaCca788fqzJl7gQicKEGMfQ/640?wx_fmt=jpeg&wxfrom=5&wx_lazy=1&wx_co=1)

可以看到，线程 A 在执行 ThreadB.start 方法之前会对共享变量进行修改，修改之后的共享变量会直接刷新到内存中，然后线程 A 执行 ThreadB.start 方法，紧接着线程 B 会从内存中读取共享变量。

## 线程 join 规则

All actions in a thread happen-before any other thread successfully returns from a join() on that thread.

这条规则是对多条线程来说的：如果线程 A 执行操作 ThreadB.join() 并成功返回，那么线程 B 中的任意操作都 happens - before 于线程 A 从 ThreadB.join 操作成功返回。

假设有两个线程 s、t，在线程 s 中调用 t.join() 方法。则线程 s 会被挂起，等待 t 线程运行结束才能恢复执行。当t.join() 成功返回时，s 线程就知道 t 线程已经结束了。所以根据本条原则，在 t 线程中对共享变量的修改，对 s 线程都是可见的。类似的还有 Thread.isAlive 方法也可以检测到一个线程是否结束。

## 线程传递规则

If an action a happens-before an action b, and b happens before an action c, then a happensbefore c.

这是 happens - before 的最后一个规则，它主要说的是操作之间的传递性，也就是说，如果 A happens-before B，且 B happens-before C，那么 A happens-before C。

线程传递规则不像上面其他规则有单独的用法，它主要是和 volatile 规则、start 规则和 join 规则一起使用。

### 和 volatile 一起使用

比如现在有四个操作：**普通写、volatile 写、volatile 读、普通读**，线程 A 执行普通写和 volatile 写，线程B 执行volatile 读和普通读，根据程序的顺序性可知，普通写 happens - before volatile 写，volatile 读 happens - before 普通读，根据 volatile 规则可知，线程的 volatile 写 happens - before volatile 读和普通读，然后根据线程传递规则可知，普通写也 happens - before 普通读。

![图片](https://mmbiz.qpic.cn/mmbiz_png/A3ibcic1Xe0iaQ3bQGCubgA1AePT7Iq1t9rO1Zh45pib8KezGwNHFaDGyU2SEYUK8ADfKodn5eAVKzGBN39JLp2j7A/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 和 start 规则一起使用

和 start 规则一起使用，其实我们在上面描述 start 规则的时候已经描述了，只不过上面那幅图少画了一条线，也就是 ThreadB.start happens - before 线程 B 读共享变量，由于 ThreadB.start 要 happens - before 线程 B 开始执行，然而从程序定义的顺序来说，线程 B 的执行 happens - before 线程 B 读共享变量，所以根据线程传递规则来说，线程 A 修改共享变量 happens - before 线程 B 读共享变量，如下图所示。

![图片](https://mmbiz.qpic.cn/mmbiz_png/A3ibcic1Xe0iaQ3bQGCubgA1AePT7Iq1t9rldGHibnUSU9R80sa5Y2eJCS3dQ7xgtORhV8lKFQJEPvYjwiaazpHmiaUA/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 和 join 规则一起使用

假设线程 A 在执行的过程中，通过执行 ThreadB.join 来等待线程 B 终止。同时，假设线程 B 在终止之前修改了一些共享变量，线程 A 从 ThreadB.join 返回后会读这些共享变量。

![图片](https://mmbiz.qpic.cn/mmbiz_png/A3ibcic1Xe0iaQ3bQGCubgA1AePT7Iq1t9rKL8fBtFF0Af1EdtOR3BOWiceBjbrKz4en0ldbaSa4CwSVwbSRALeNQQ/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)



在上图中，2 happens - before 4 由 join 规则来产生，4 happens - before 5 是程序顺序规则，所以根据线程传递规则，将会有 2 happens - before 5，这也意味着，线程 A 执行操作 ThreadB.join 并成功返回后，线程 B 中的任意操作将对线程 A 可见。


