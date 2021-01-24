---
title: ReentrantLock原理分析
author: 旺旺
date: 2021-01-10 21:48:00 +0800
categories: [Java, JUC]
tags: [Java, JUC, AQS, Lock, ReentrantLock]
---

ReentrantLock是Java并发包中提供的一个可重入的互斥锁，它拥有与synchronized相同的作用，但却比synchronized有更好的性能，在许多高并发编程中都会用到它。由于大部分同学都只停留在了API调用的层次，对ReentrantLock的原理一知半解，甚至一无所知，因此写下了这篇文章，让同学们真正的把ReentrantLock给拿下！

本文将会从以下几个方面去进行分享：

* 使用场景
* 源码实现
* 设计思想

### 使用场景

```java
public class ReentrantLockTest {
    private ReentrantLock lock = new ReentrantLock();

    public void method() {
        lock.lock();
        // do something
        lock.unlock();
    }
}
```

ReentrantLock的使用十分简单，在同步代码块前调用lock()加锁，同步代码块之后调用unlock()释放锁就可以了。另外要注意，lock()和unlock()必须成双成对的出现。如果同步代码块可能抛出异常，则必须把unlock()调用放在finally块里。

### 源码实现

打开lock()查看它的实现。

```java
public void lock() {
    sync.lock();
}
```

它通过调用了sync的lock()方法来完成加锁，我们去看下sync的定义。

```java
private final ReentrantLock.Sync sync;
```

Sync是一个内部类，我们去看下它的lock()实现。

```java
abstract void lock();
```

很明显，有子类继承了Sync，这时候我们可以去看sync的初始化代码，看看是使用了哪个子类对sync进行了初始化。

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

有两个子类用于sync的初始化，FairSync和NonfairSync。这其实就是我们所熟知的公平锁和非公平锁。ReentrantLock默认情况下使用了非公平锁，当然也可以在创建ReentrantLock的时候显示指定。

现在我们先去看下非公平锁NonfairSync对lock()的实现。

```java
final void lock() {
    // 这步快速尝试获取锁的操作，公平锁里边没有
    if (this.compareAndSetState(0, 1)) {
        this.setExclusiveOwnerThread(Thread.currentThread());
    } else {
        this.acquire(1);
    }
}
```

查看代码可知，非公平锁一上来就先调用一把compareAndSetState()，尝试获取锁，这个对于已经在锁队列里苦苦等待的其他线程，是非常不公平的。划重点了，同学们，这里是公平锁和非公平锁的重要区别。

现在我们来看compareAndSetState()的实现。

```java
protected final boolean compareAndSetState(int var1, int var2) {
    return unsafe.compareAndSwapInt(this, stateOffset, var1, var2);
}
```

原理是通过unsafe提供的CAS原子操作进行state的值更新。另外发现compareAndSetState()是位于AbstractQueuedSynchronizer类中的，继而发现，Sync继承了AbstractQueuedSynchronizer，我们需要更新的state也位于AbstractQueuedSynchronizer中。

```java
private volatile int state;
```

state记录了锁重入的次数，如果为0，那么表示当前没有线程持有此锁，此时使用一个CAS操作即可快速完成锁的申请，这便是快速尝试。

当快速尝试失败之后，将会调用acquire()方法，acquire()也是来自于AbstractQueuedSynchronizer，我们看下代码。

```java
public final void acquire(int var1) {
    if (!this.tryAcquire(var1) && this.acquireQueued(this.addWaiter(Node.EXCLUSIVE), var1)) {
        selfInterrupt();
    }
}
```

先tryAcquire()一下，万一自己是个锁二代（锁重入）呢，那就爽歪歪了，获取锁成功，直接撤退走人！

来看下非公平锁是怎么获取锁的，打开tryAcquire()的源码。

```java
protected final boolean tryAcquire(int var1) {
    return this.nonfairTryAcquire(var1);
}
```

继续查看Sync中的nonfairTryAcquire()。

```java
final boolean nonfairTryAcquire(int var1) {
    // 获取当前线程
    Thread var2 = Thread.currentThread();
    // 获取锁重人次数
    int var3 = this.getState();
    // lock还没有被任何线程霸占，赶紧快速尝试一把加锁
    if (var3 == 0) {
        if (this.compareAndSetState(0, var1)) {
            this.setExclusiveOwnerThread(var2);
            return true;
        }
    } 
    // lock已经被线程霸占了，检查一下是不是自己人，如果是的话，那当前线程就是锁二代了，state加1
    else if (var2 == this.getExclusiveOwnerThread()) {
        int var4 = var3 + var1;
        // 锁重入的次数不能超过Ingteger.MAX_VALUE，不然会爆炸
        if (var4 < 0) {
            throw new Error("Maximum lock count exceeded");
        }
        this.setState(var4);
        return true;
    }
	// 既没有创业成功，也不是锁二代，就只有失败的命运了
    return false;
}
```

看完nonfairTryAcquire()的操作，我们知道非公平锁的锁重入是怎么玩的了。

如果线程没有获取到锁，就只能去队列里等待锁了，也就是调用addWaiter()方法，我们来看下它的实现。

```java
private Node addWaiter(Node mode) {
    // 构造一个Node，与当前线程绑定，mode传入的是Node.EXCLUSIVE，代表独占锁
    Node var2 = new Node(Thread.currentThread(), mode);
    // 获取队列末尾节点
    Node var3 = this.tail;
    // 末尾节点非空，CAS快速尝试，把自己更新为末尾节点
    if (var3 != null) {
        var2.prev = var3;
        if (this.compareAndSetTail(var3, var2)) {
            var3.next = var2;
            return var2;
        }
    }
    // 末尾节点不存在，或者更新末尾节点失败了
    this.enq(var2);
    return var2;
}
```

当末尾节点为null，或者更新末尾节点失败了，那就调用enq()进行处理。

```java
private Node enq(Node var1) {
    // 注意这里的while(true)，不达目的不罢休
    while(true) {
        Node var2 = this.tail;
        // 末尾节点为空，意味着整个队列都为空，头节点自然不存在，那就来初始化一波头尾节点
        if (var2 == null) {
            // 通过CAS更新头节点，从这行代码我们也可以知道，锁队列里的头节点是空的，没有和任何线程绑定
            if (this.compareAndSetHead(new Node())) {
                // 此时头节点和末尾节点是同一个
                this.tail = this.head;
            }
        } 
        // 末尾节点已经存在，直接CAS把自己更新为末尾节点
        else {
            var1.prev = var2;
            if (this.compareAndSetTail(var2, var1)) {
                var2.next = var1;
                return var2;
            }
        }
    }
}
```

用上了while(true)，保证了enq()返回后，当前线程一定是被加入到了锁队列的末尾。

当前线程对应的Node加入队列末尾之后，接着调用了acquireQueued()，我们来看下这个方法干了什么事。

```java
final boolean acquireQueued(Node var1, int var2) {
    // 标识该方法返回时，当前线程是否已获得锁，默认值true代表没有抢到
    boolean var3 = true;

    try {
        // 标识一下当前线程在睡觉时候有没有被叫醒过
        boolean var4 = false;
		// 自旋获取锁
        while(true) {
        	// 获取当前节点的上一个节点 
            Node var5 = var1.predecessor();
            // 如果上一个节点是头节点的话，就可以直接尝试抢锁
            if (var5 == this.head && this.tryAcquire(var2)) {
                //把自己设置为头节点
                this.setHead(var1);
                // 解除上一任头节点的依赖，让它早日被GC干掉
                var5.next = null;
                // 标识我已经抢锁成功啦
                var3 = false;
                // 最终的返回值，居然是当前线程睡觉时候有没有被叫醒过
                boolean var6 = var4;
                return var6;
            }
			// 当前节点不在头节点之后，或者在头节点之后，但是抢锁失败了
            // 调用shouldParkAfterFailedAcquire()，为自己找到一个归宿(让上一个节点完事之后通知自己)，然后就可以调用parkAndCheckInterrupt()让自己去休眠了
            if (shouldParkAfterFailedAcquire(var5, var1) && this.parkAndCheckInterrupt()) {					
                // 睡觉时被意外唤醒，记录一下，自己也是发生过中断的男人了
                var4 = true;
            }
        }
        
    } finally {
        // 如果var3为true，则证明线程没有拿到锁，并且它已经废了，所以方法退出前，得调用cancelAcquire()给线程收尸
        if (var3) {
            this.cancelAcquire(var1);
        }
    }
}
```

线程被加入到队列之后，就是疯狂自旋的干上面这几件事情：找人叫醒自己，睡觉，被叫醒，周而复始，直到自己拿到了锁，然后离开。

至于线程怎么找人叫醒自己的，我们来看shouldParkAfterFailedAcquire()的实现。

```java
// var0是上一个节点，var1是当前节点
private static boolean shouldParkAfterFailedAcquire(Node var0, Node var1) {
    int var2 = var0.waitStatus;
    // 上一个节点满足被叫醒的条件，那也就意味着上一个节点早晚会抢锁，用完锁后自然会通知自己，这样的话，自己就可以安心去睡觉了
    if (var2 == -1) {
        return true;
    } else {
        // 上一个节点放弃抢锁啦，指望不上了，继续往前寻找可靠的节点作为依靠
        if (var2 > 0) {
            do {
                var1.prev = var0 = var0.prev;
            } while(var0.waitStatus > 0);
			
            var0.next = var1;
        } else {	// waitStatus不大于0，CAS把它设置为-1（满足被唤醒的条件），但是设置不一定会成功
            compareAndSetWaitStatus(var0, var2, -1);
        }
		// 这一波操作，没有找到唤醒自己的人，睡不成啰
        return false;
    }
}
```

既然睡不成，那还是继续去看看有没有抢锁资格吧，有就抢一把，就这样周而复始的的循环下去。

当某一时刻，线程找到了能叫醒自己的人，这时候它就可以去睡觉了，去睡觉自然就是调用parkAndCheckInterrupt()方法。

```java
private final boolean parkAndCheckInterrupt() {
    // 划重点了，同学们，线程阻塞就是调用这个API来完成的，底层的实现是用的unsafe.park()
    LockSupport.park(this);
    // 线程睡醒了，但是它要判断一下睡觉期间有没有发生过中断
    return Thread.interrupted();
}
```

如果发生过中断，则parkAndCheckInterrupt()会返回true。这是我们再去看acquire()方法，它会执行selfInterrupt()。

```java
static void selfInterrupt() {
    // 给线程标记上中断位，这可谓中断会延迟处理，但是从未缺席
    Thread.currentThread().interrupt();
}
```

同学们，到这里lock()就分析完了。现在我们接着来看看unlock()是怎么玩的。

```java
public void unlock() {
    this.sync.release(1);
}
```

看样子是调用了AQS的release()方法，我们接着看。

```java
public final boolean release(int var1) {
    // 释放锁，只有state变为0了才会返回true
    if (this.tryRelease(var1)) {
        Node var2 = this.head;
        // 头节点不为空，代表队列不为空，waitStatus不为0，代表它有后继节点，因此可以去唤醒下家去抢锁
        if (var2 != null && var2.waitStatus != 0) {
            this.unparkSuccessor(var2);
        }
        return true;
    } else {
        return false;
    }
}
```

调用tryRelease()方法释放锁，看下它的实现。

```java
protected final boolean tryRelease(int var1) {
    // state减1后的结果
    int var2 = this.getState() - var1;
    // 如果线程不是当前锁的线程，那就玩大啦，吃不了逗着走，直接抛出异常
    if (Thread.currentThread() != this.getExclusiveOwnerThread()) {
        throw new IllegalMonitorStateException();
    } else {
        // 标识锁是不是已经完全释放了
        boolean var3 = false;
        // 没有线程占用锁了，可以让下一个线程来持锁了
        if (var2 == 0) {
            // 锁完全释放了就返回true
            var3 = true;
            // 把锁的持有者设置为null
            this.setExclusiveOwnerThread((Thread)null);
        }
		// 更新state值
        this.setState(var2);
        return var3;
    }
}
```

如果锁完全释放了，那么就得唤醒下家去抢锁。具体是怎么寻找下家的呢，看一下unparkSuccessor()。

```java
private void unparkSuccessor(Node var1) {
    int var2 = var1.waitStatus;
    if (var2 < 0) {
        // 将头节点设置为初始状态
        compareAndSetWaitStatus(var1, var2, 0);
    }

    Node var3 = var1.next;
    if (var3 == null || var3.waitStatus > 0) {
        var3 = null;
		// 从队列的末尾节点往前找下家，最终是找到队列里(头节点除外)最前面的节点，作为唤醒对象
        for(Node var4 = this.tail; var4 != null && var4 != var1; var4 = var4.prev) {
            if (var4.waitStatus <= 0) {
                var3 = var4;
            }
        }
    }
	// 唤醒这个节点
    if (var3 != null) {
        LockSupport.unpark(var3.thread);
    }
}
```

非公平锁到这里就讲完了，至于tryLock()方法，相信同学们在看完为上面lock()的分享，已经可以自己独立把它拿下了，现在我们来讲一下公平锁。前面已经提到了公平锁和非公平锁的一个区别，就是lock()里的tryAcquire()实现有所不同。非公平锁任何一个新加入的线程都可以参与抢锁，但是公平锁就得老老实实排队，讲究个先来后到，具体来看下吧。

```java
protected final boolean tryAcquire(int var1) {
    Thread var2 = Thread.currentThread();
    int var3 = this.getState();
    if (var3 == 0) {
        // hasQueuedPredecessors()很关键，它是公平性的核心体现
        if (!this.hasQueuedPredecessors() && this.compareAndSetState(0, var1)) {
            this.setExclusiveOwnerThread(var2);
            return true;
        }
    } else if (var2 == this.getExclusiveOwnerThread()) {
        // 锁重入
        int var4 = var3 + var1;
        if (var4 < 0) {
            throw new Error("Maximum lock count exceeded");
        }
        this.setState(var4);
        return true;
    }
	// 抢锁失败了
    return false;
}
```

hasQueuedPredecessors()的作用是当满足以下两种条件中的一种时，线程就能获得抢锁的资格: 1. 锁同步队列里只有一个节点；2. 第二个节点属于当前线程。

### 设计思想

先看一下AQS内部维护的锁同步队列。

![aqs-queue](http://www.giver.vip/article_image/aqs-queue.png)

ReentrantLock通过使用AQS来实现加解锁。AQS内部维护了一个双向链表的锁同步队列，并维护头节点head，尾节点tail和信号量state。每个节点是一个Node对象，对象中定义了prev，next分别指向它的上下游，还有一个waitStatus对象用于表示线程状态(等锁或已放弃)。当有新的线程需要抢锁时，新建一个和线程映射的Node，加入到锁同步队列的末尾。当然这里有个重点，在加入的时候会做判断，如果当前末尾节点处于放弃状态，那么会继续往前遍历，寻找一个可靠的节点作为上游。AQS内部的state为0时，资源未被占用，线程可进行CAS操作更新state，如果更新成功则代表加锁成功。如果state不为0，则意味着资源已经被线程占用。如果占用者是自己，那么可以进行重入，如果占用者不是自己，那么就老老实实等着。

关于ReentrantLock的源码讲解和原理分析，到这里就全部结束啦。后续还会更新更多关于Java并发包的其他干货，同学们一定要结合起来阅读，相辅相成，形成一个完整的知识体系。

小瑾的heart，就像那AQS的state。state可以共享，而heart只能独占。而我，就是那个独占了heart的线程。现在，我死循环了，将一辈子不会释放heart。