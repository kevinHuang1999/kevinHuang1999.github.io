---
layout:     post   				    # 使用的布局（不需要改）
title:      从ReentranLock到AQS(1) 				# 标题 
subtitle:   java-jdk-JUC #副标题
date:       2020-11-09 				# 时间
author:     BY 	KevinHuang					# 作者
header-img: img/post-bg-2020-11-09-1.jpg 	#这篇文章标题背景图片
catalog: true 						# 是否归档
tags:								#标签
    - Java
    - jdk
    - juc





---

> ReentrantLock是juc包下实现Lock接口的可重入锁，在早期的jdk版本中，ReentrantLock的性能要好于synchronized，在jdk6开始，jdk对synchronized做了大量的的优化，使得两者性能的差距不大。

最近一直在学习juc，学到锁的底层发现里面的东西是比较复杂的（头大），我也来慢慢地总结我学习到的东西。

### 1.类注释

```java
/**
 * A reentrant mutual exclusion {@link Lock} with the same basic
 * behavior and semantics as the implicit monitor lock accessed using
 * {@code synchronized} methods and statements, but with extended
 * capabilities.
 * 可重入互斥锁，和synchronized有相同的功能语义,但更具有拓展性
 **/
```

----------------------

```java
/*
* <p>The constructor for this class accepts an optional
* <em>fairness</em> parameter.
* .....
* 构造器可以接受fairness参数，true为公平锁,false为不公平锁
*/
```

------

```java
/*
* Programs using fair locks accessed by many threads
* may display lower overall throughput (i.e., are slower; often much
* slower) than those using the default setting, but have smaller
* variances in times to obtain locks and guarantee lack of
* starvation.
* 公平锁比默认的锁(非公平)的吞吐量低，但是在获得锁的时间上有较小的差异，并保证不会产* 生饥饿 
*/
```

----

```java
/*
* Also note that the untimed {@link #tryLock()} method does not
* honor the fairness setting. It will succeed if the lock
* is available even if other threads are waiting.
* tryLock()无参方法没有遵循公平性，
*/
```



### 2.类结构

ReentrantLock实现了Lock的接口

```java
public class ReentrantLock implements Lock, java.io.Serializable
```

Lock 接口定义了各种加锁，释放锁的方法，也是我们常用的方法：

```java
// 获得锁方法，获取不到锁的线程会到同步队列中阻塞排队
void lock();
// 获取可中断的锁
void lockInterruptibly() throws InterruptedException;
// 尝试获得锁，如果锁空闲，立马返回 true，否则返回 false
boolean tryLock();
// 带有超时等待时间的锁，如果超时时间到了，仍然没有获得锁，返回 false
boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
// 释放锁
void unlock();
// 得到新的 Condition
Condition newCondition();
```

ReentrantLock负责实现这些接口，而这些方法的底层实现都是交给Sync内部类去实现，而Sync继承了AbstractQueuedSynchronizer，也就是AQS

```java
abstract static class Sync extends AbstractQueuedSynchronizer
```

Sync只需要实现AQS预留的几个方法即可，但 Sync 也只是实现了部分方法，还有一些交给子类 NonfairSync 和 FairSync 去实现了，NonfairSync 是非公平锁，FairSync 是公平锁

```java
static final class FairSync extends Sync {}
static final class NonfairSync extends Sync {}
```

![](https://github-blog-kevinhuang-1304012368.cos.ap-shanghai.myqcloud.com/img/20201109160352.png)

结构如上图

### 3.构造器

```java
// 无参数构造器，相当于 ReentrantLock(false)，默认是非公平的
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

### 4.Sync同步器

![](https://github-blog-kevinhuang-1304012368.cos.ap-shanghai.myqcloud.com/img/20201109160953.png)

可以看出，lock 方法是个抽象方法，留给 FairSync 和 NonfairSync 两个子类去实现。

### 5.AQS

在之前我看到这时候，AQS到底是个什么东西？基本上在juc包下的类都能看到它的身影。

AQS的全称是AbstractQueuedSynchronizer，译名抽象队列同步器。可以说是整个Java并发编程包JUC的底层实现，AQS是一种采用CAS+自旋的方式，然后采用一个双向链表组成的队列，结合一个state来实现的同步器。

看看源码的注释，而这个同步的队列作者称为CLH

```java
/*
* <p>To enqueue into a CLH lock, you atomically splice it in as new
* tail. To dequeue, you just set the head field.
* <pre>
*      +------+  prev +-----+       +-----+
* head |      | <---- |     | <---- |     |  tail
*      +------+       +-----+       +-----+
* </pre>
*/
```

一个锁是否被抢占到使用state来标记的，而state是使用volatile修饰的，可以对所以线程保持可见性。当某个线程抢占到了锁，会将state设为1，然后其他没抢占到的线程会被封装成一个node添加进同步队列中，等待持有锁的线程释放锁后，会唤醒离头结点最近的节点的线程，继续抢占锁。整个过程是基于CAS来实现的，保证同时只有一个线程能修改state值。

### 6. AQS 属性

#### 6.1 简单属性

```java
// 同步器的状态，子类会根据状态字段进行判断是否可以获得锁
// 比如 CAS 成功给 state 赋值 1 算得到锁，赋值失败为得不到锁， CAS 成功给 state 赋值 0 算释放锁，赋值失败为释放失败
// 可重入锁，每次获得锁 +1，每次释放锁 -1
private volatile int state;
```

#### 6.2 同步队列属性

```java
// 同步队列的头。
private transient volatile Node head;

// 同步队列的尾
private transient volatile Node tail;
```

关于条件队列，下次我再总结

#### 6.3 节点NODE

```java
static final class Node {
    /**
     * 同步队列单独的属性
     */
    //node 是共享模式
    static final Node SHARED = new Node();
    //node 是排它模式
    static final Node EXCLUSIVE = null;

    // 当前节点的前节点
    // 节点 acquire 成功后就会变成head
    // head 节点不能被 cancelled
    volatile Node prev;
    // 当前节点的下一个节点
    volatile Node next;
		// 当前节点的线程
    volatile Thread thread;
  
    // 表示当前节点的状态，通过节点的状态来控制节点的行为
    // 普通同步节点，就是 0
    volatile int waitStatus;

    // waitStatus 的状态有以下几种
    // 被取消
    static final int CANCELLED =  1;

    // SIGNAL 状态的意义：同步队列中的节点在自旋获取锁的时候，如果前一个节点的状态是 SIGNAL，那么自己就可以阻塞休息了，否则自己一直自旋尝试获得锁
    static final int SIGNAL    = -1;

    // 表示当前 node 正在条件队列中，当有节点从同步队列转移到条件队列时，状态就会被更改成 CONDITION
    static final int CONDITION = -2;

    // 无条件传播,共享模式下，该状态的进程处于可运行状态
    static final int PROPAGATE = -3;

    // 在同步队列中，nextWaiter 并不真的是指向其下一个节点，我们用 next 表示同步队列的下一个节点，nextWaiter 只是表示当前 Node 是排它模式还是共享模式
    // 但在条件队列中，nextWaiter 就是表示下一个节点元素
    Node nextWaiter;
}
```

### 7.获取锁

先大概看下获取锁的流程图

![](https://github-blog-kevinhuang-1304012368.cos.ap-shanghai.myqcloud.com/img/image-20201109212614865.png)

我们可以从公平锁FairSync来看看线程是如何获取锁的

```java
final void lock() {
    acquire(1);
}
```

```java
//tryAcquire是AQS的模板方法，留给子类实现
public final void acquire(int arg) {
  //先CAS尝试获取锁，如果获取失败则将node添加至同步队列中  
  if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();//自我中断，当获取锁的时候，发生中断时记录下来，推迟到抢锁结束后中断线程
}
```

下面看看FairSync 的 tryAcquire() 尝试获取锁的方法

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread(); //获取当前线程
    int c = getState(); //得到当前的state
    if (c == 0) { //如果state未被占用，尝试获取锁
      	//先判断链表有没有前面排队的节点，若没有
      	//再去使用CAS去尝试修改state，成功则设
      	//置该线程占用该锁
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
  	//若想获取锁的线程为持有该锁的线程，则可以重入
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires; //state+1
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc); 
        return true;
    }
    return false;
}
```

再来看看 **acquireQueued(addWaiter(Node.EXCLUSIVE), arg)** 这个方法，里面的**addWaiter(Node mode)**是由AQS实现的，目的是将线程封装成Node加入同步队列。而

**acquireQueued(final Node node, int arg)** 方法是查看刚刚的节点的前驱是否是头节点，是就自旋获取锁，成功就返回。如果其前驱不是头结点，或是头结点但是获取失败，挂起当前Node代表的线程。

```java
//false表示获取锁成功，true表示失败
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
      	//自旋
        for (;;) {
          	//前驱节点
            final Node p = node.predecessor();
          	//前驱是头节点
          	//1.如果自己 tryAcquire 成功，就立马把自己设置成 head，
          	//把上一个节点移除
            //2.如果 tryAcquire 失败，尝试进入同步队列
            if (p == head && tryAcquire(arg)) {
                setHead(node); //设置为头节点
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
          	// shouldParkAfterFailedAcquire 
          	// 把node 的前一个节点状态置为 SIGNAL -1
            // 只要前一个节点状态是 SIGNAL了，那么自己就可以阻塞(park)了
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 线程是在这个方法里面阻塞的，醒来的时候仍然在无限 for 循环里								  //面，就能再次自旋尝试获得锁
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
      	// 如果获得node的锁失败，将 node 从队列中移除
        if (failed)
            cancelAcquire(node);
    }
}
```

接下来看**shouldParkAfterFailedAcquire**这个方法，用于将前一个节点状态置为SIGNAL，

这样此节点就可以park了。

```java
//判断acuire失败后是否阻塞
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; 
  	//前一节点是SIGNAL，直接返回
    if (ws == Node.SIGNAL)
        return true;
  	//前一个节点已经被取消了
    if (ws > 0) {
        do {
          	//将有效节点挂载到node.prev上
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    //否则将前节点状态置为SIGNAL，使用CAS
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

再回去看看**addWatier**这个方法，作用是将node添加进同步队列，主要思路和链表插入的方法差不多，新的节点node.pre = tail ，tail.next = node

```java
private Node addWaiter(Node mode) {
  	//初始化
    Node node = new Node(Thread.currentThread(), mode);
    Node pred = tail;
  	//tail不为空
    //在这里先进行了一次简单的CAS，成功立马返回节点，不成功再进行自旋
  	//这也是jdk中 "先尝试放一次" 的想法，可以减少自旋次数
  	if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
  	//自旋加入
    enq(node);
    return node;
}
```

```java
//添加到队尾
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
          //初始化一个哨兵节点(空)作为头节点  
          if (compareAndSetHead(new Node()))
                tail = head;
        } else { //队尾不为空
            node.prev = t;
          	//node追加到队尾
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

可以知道，acquire大致分为三步

```java
1.使用tryAcquire方式尝试获取锁，获取锁直接返回，失败进入2

2.把当前线程封装成node，加入同步队列的队尾

3.自旋，将同步队列的前置节点状态置为SIGNAL后，然后park自己
```

还有一个关键点就是，为什么头节点指向的是**哨兵节点**？

​	可以这样去理解，头节点是不参与排队的，也就是拿到锁线程的节点，因为它已经取得同步状态了。而在执行完业务逻辑，释放同步状态后，该头节点是肯定要被垃圾回收的，防止内存空间的浪费，如果对象还有引用的话，垃圾回收器是不会回收它的，所以需要把头节点持有的各种引用都置为null，方便之后的垃圾回收，而Thread也是头节点持有的引用之一，因此Thread对象也需要被置为null。

而在第一次入队时，只能增加哨兵

可以参考下图过程：

![image-20201109215604079](https://github-blog-kevinhuang-1304012368.cos.ap-shanghai.myqcloud.com/img/image-20201109215604079.png)

![image-20201109215842526](https://github-blog-kevinhuang-1304012368.cos.ap-shanghai.myqcloud.com/img/image-20201109215842526.png)



参考：

https://www.imooc.com/read/47/article/872

https://www.jianshu.com/p/e0d4bc1408f8

https://blog.csdn.net/weixin_38106322/article/details/107141976