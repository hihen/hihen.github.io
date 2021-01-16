---
title: volatile实现原理
author: 旺旺
date: 2020-09-15 21:30:00 +0800
categories: [Java, JVM]
tags: [Java, JVM, volatile]
---

最近学习了volatile的实现原理，有一些心得体会，写这篇文章记录一下。

将会从以下几个方面去描述volatile的技术内幕:
* 功能特性与使用场景
* 字节码层面分析
* JVM层面分析
* CPU层面分析

### 功能特性与使用场景

#### 保证数据可见性
在多线程环境下，多个线程共享某个volatile修饰的数据。当其中一个线程修改了这个数据之后，其他线程将能及时获取到该数据的最新值。

为什么当一个线程修改了volatile修饰的数据之后，其他线程能及时获取到该数据的最新值呢？可以通过下面这张图得出答案。

![java-concurrent-memory-model](http://www.giver.vip/article_image/java-concurrent-memory-model.jpg)

上图的java并发内存模型并不是真实存在于计算机中的，只是抽象出来的一个概念，具体实现依赖底层操作系统和硬件。图中三个线程拥有自己的独立工作内存，用于存放线程运行需要的数据。假如三个线程都将主存里一个被volatile修饰的数据a读取到自己工作内存，这个时候线程1对工作内存中的数据a作出修改并回写到主存，线程2和线程3对应工作内存中的数据a会同步的变成无效状态。当线程2或线程3准备使用数据a时，发现是无效状态，将重新去上一级缓存中读取一个数据a进来，此时所有缓存里的数据a都是最新值，如此保证了数据的一致性。

使用场景:
```java
public class VolatileVisibleTest {
    private volatile boolean flag = true;

    public void startWork() {
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (flag) {
                    // do something
                }
            }
        }).start();
    }

    public void endWork() {
        flag = false;
    }
}
```

#### 禁止指令重排序
使用volatile修饰的某个数据，操作该数据的指令将不会进行重排序。在知道volatile能够禁止指令重排序的同时，需要思考一个问题: 为什么会出现指令重排序？

学过计算机组成原理的都知道，计算机中存储设备主要可以分为以下几类:寄存器，L1高速缓存，L2高速缓存，L3高速缓存，主存，本地磁盘，远程存储设备，排序越往后的读取速度越慢。假设某一条指令是将计算好的a值写回到主存，需要花费100ms，而这条指令之后的下一条指令，是读取L1高速缓存中的数据b到寄存器，只需要花费1ms，两条指令谁先执行谁后执行，对最终结果都没有影响，那么这个时候cpu为了提高计算效率，就会出现指令重排序的情况。理论是这么说的，但在真实的计算机环境中，cpu真的会进行指令重排序吗？ 带着这样的怀疑，我特意做了一个小试验，验证了cpu确实会出现指令重排序。对验证过程感兴趣的可以移步到这篇文章[一种证明cpu重排序的最简单方式]()

使用场景:
```java
public class VolatileBanOutOfOrder {
    private static volatile VolatileBanOutOfOrder singleObject;

    private VolatileBanOutOfOrder() {
    }

    public static VolatileBanOutOfOrder getSingleObject() {
        if (singleObject == null) {
            synchronized (VolatileBanOutOfOrder.class) {
                if (singleObject == null) {
                    singleObject = new VolatileBanOutOfOrder();
                }
            }
        }
        return singleObject;
    }
}
```

### 字节码层面分析
在下图中，左边部分为java源代码，右边为编译后的字节码信息。从字节码信息中可以看出，该类有一个属性b，重点看属性b描述信息中Access flags这个信息，它记录了属性b被volatile修饰。其实说白了就是在字节码层面有一个volatile去放在属性b的Access flags中就OK了。想了解真正的实现细节，还是要看JVM层面和硬件层面的分析。

![volatile](http://www.giver.vip/article_image/volatile.png)

### JVM层面分析

JVM层面在处理volatile相关的指令时，会使用到内存屏障(Memory Barrier)。什么是内存屏障？先来看下JVM规范中对内存屏障的定义：

```terminal
LoadLoad屏障：
对于这样的语句Load1；LoadLoad；Load2，
在Load2及后续读取操作执行之前，LoadLoad保证了Load1读取操作执行完毕。
```

```terminal
StoreStore屏障：
对于这样的语句Store1；StoreStore；Store2，
在Store2及后续写入操作执行之前，StoreStore保证了Store1写入操作执行完毕。
```

```terminal
StoreLoad屏障：
对于这样的语句Store1；StoreLoad；Load2，
在Load2及后续读取操作执行之前，StoreLoad保证了Store1写入操作执行完毕，执行结果对所有处理器可见。
```

```terminal
LoadStore屏障：
对于这样的语句Load1；LoadStore；Store2，
在Store2及后续写入操作执行之前，LoadStore保证了Load1读取操作执行完毕。
```

了解了什么是内存屏障后，再来看JVM是怎么运用内存屏障来处理volatile的。
```terminal
StoreStoreBarrier
volatile写操作
StoreLoadBarrier
```

```terminal
LoadLoadBarrier
volatile读操作
LoadStoreBarrier
```

现在，终于把防止指令重排序的原理弄清楚了。想知道volatile保证数据一致性的原理，接着看CPU层面的分析。

### CPU层面分析

在学习CPU层面保证数据一致性的原理之前，需要先学习两个知识。一个是CPU执行计算的过程，在这里我简单的总结了一下，过程如下：
```termanal
1. 程序以及数据被加载到主内存
2. 指令加载到高速缓存和CPU缓存
3. 数据加载到高速缓存和CPU缓存
4. CPU执行指令，把结果写到高速缓存
5. 高速缓存中的数据回写到主存
```
另外一个知识是CPU的缓存一致性协议。早期的CPU只有一个内存总线锁，锁的力度和开销都很大，后来有了缓存一致性协议，诞生了缓存锁，力度和开销远远小于总线锁，CPU的处理效率得到很大提升。发展至今出现了许多缓存一致性协议，其中比较主流的是intel CPU的MESI协议。在这里就不展开了，大家感兴趣可以自行去学习。

我们来看CPU层面volatile保证数据一致性的原理实现：
如果多个不同的CPU都读取了同一个数据到自己的高速缓存，这时候数据属于共享状态。其中一个CPU对数据进行了计算并准备回写，回写之前它会发送一个lock指令，lock指令会干两件事情，一件事情是把CPU共享缓存中该数据的缓存进行锁定，另一件事情是通知其他CPU把它们内部的缓存数据设为失效状态。lock指令执行之后，CPU开始回写数据到高速缓存。如果这过程中其他CPU需要读取这个数据，发现自己内部缓存是失效状态，则会去共享缓存读取，结果发现共享缓存中该数据是锁定状态，那么它只能等待锁释放。当回写数据执行完毕后，锁释放，其他CPU将从共享缓存中读取到最新的数据。此时，所有CPU缓存中数据是一致的。

说到这里，volatile的原理就算分析完了。最后还得提醒一个很容易犯错误的地方: volatile能保证多线程环境中的数据可见性，但无法保证线程安全。许多同学把这两个概念混淆了，甚至认为volatile和synchronized是差不多一样的东西，要纠正一下这个错误认知。

终于写完了，旁边的凤爪也快凉了。不过凉了也得吃啊，毕竟是小瑾亲自买的，通通吃完！

