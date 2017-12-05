---
title: AQS的基本原理
tags:
  - Java
  - 并发
  - 原创
reward: true
date: 2017-12-04 20:38:02
---

AQS（AbstractQueuedSynchronizer）是一个用于构建锁和同步器的框架，许多同步器都可以通过AQS很容易并且高效的构造出来。不仅Reentrant和Semaphore是基于AQS构建的，还包括CountDownLatch、ReentrantReadWriteLock、SynchronousQueue和FutureTask。

<!--more-->

# 高层抽象

在基于AQS构建的同步器类中，最基本的操作包括各种形式的获取操作和释放操作。

**获取操作是一种依赖状态的操作，并且通常会阻塞直到达到了理想状态**。如：

* 使用ReentrantLock时，“获取”操作意味着“等待直到锁可被获取”。
* 使用Semaphore时，“获取”操作意味着“等待直到许可可被获取”。
* 使用CountDownLatch时，“获取”操作意味着“等待直到闭锁达到结束状态”。
* 使用FutureTask时，“获取”操作意味着“等待直到任务完成”。

释放并不是一个可阻塞的操作，当执行“释放”操作时，所有在请求时被阻塞的线程都会开始执行。

# 内部原理

## 状态机

AQS提供了一个高效的状态机模型，用来管理同步器类中的状态和。

### 状态

AQS使用一个整数state以表示状态，并通过getState、setState及compareAndSetState等protected类型方法进行状态转换。巧妙的使用state，可以表示任何状态，如：

* ReentrantLock用state表示所有者线程已经重复获取该锁的次数。
* Semaphore用state表示剩余的许可数量。
* CountDownLatch用state表示闭锁的状态，如关闭、打开。
* FutureTask用state表示任务的状态，如尚未开始、正在运行、已完成、已取消。

除了state，在同步器类中还可以自行管理一些额外的状态变量。如：

* ReentrantLock保存了锁的当前所有者的信息，这样就能区分某个获取操作是重入的还是竞争的。
* FutureTask用result表示任务的结果，该结果可能是计算得到的答案，也可能是抛出的异常。

### 状态转换

状态转换则表现为不同的获取操作和释放操作，其标准形式如下：

```java
boolean acquire () throws InterruptedException {
  while (当前状态不允许获取操作) {
    if (需要阻塞获取请求) {
      如果当前线程不在队列中，则将其插入队列
      阻塞当前新城
    }
    else
      返回失败
  }
  可能更新同步器的状态
  如果当前线程在队列中，则将其移出队列
  返回成功
}

void release () {
  更新同步器的状态
  if (新的状态允许某个被阻塞的线程获取成功)
    接触队列中一个或多个线程的阻塞状态
}
```

为什么10行只是“可能”，而不是“必然”更新同步器的状态呢？因为获取同步器的某个线程可能对其他线程能否也获取该同步器造成影响，也可能不影响。如使用独占的ReentrantLock时，一个线程获取锁后，其他线程就不能再获取锁，于是需要更新同步器的状态；但使用CountDownLatch时，一个线程获取闭锁时（包括正在获取和获取后），不会影响其他线程能否获取它，因此不需要更新同步器的状态。

# 一些约定

根据是否支持阻塞、是否支持独占等，获取操作和释放操作都有多个实现。获取操作有acquire、acquireShared、tryAcquire、tryAcquireShared等，释放操作有release、releaseShared、tryRelease、tryReleaseShared等。

>还有acquireNanos、acquireInterruptibly等实现，为了讲解方便，暂时忽略它们。

不带try前缀的方法是阻塞的（当然release、releaseShared不是可阻塞的），通过调用带try前缀的相应版本实现，如acquire内部调用tryAcquire并维护相关逻辑。_AQS抽象类中提供了不带try前缀的方法，并以final修饰，在实现同步器时应直接使用；需要覆写的是带try前缀的方法_。对于这些方法，**约定通过返回值告知调用者（一般是AQS）获取或释放操作是否成功，一些特殊的值代表额外的信息**：

* 对于tryAcquire，如果返回true，则表示获取成功；否则返回false。
* 对于tryAcquireShared，如果返回一个负值，那么表示获取操作失败，返回零值表示同步器通过独占方式被获取，返回正值表示同步器通过非独占方式被获取。
* 对于tryRelease与tryReleaseShared方法来说，如果返回true，则表示已完全释放，所有在获取同步器时被阻塞的线程都可以被恢复执行；否则返回false。

要想基于AQS构建同步器，就必须对上述四个方法烂熟于心。

# 总结

* state表示状态，其他状态需要自行维护
* 直接使用不带try前缀的方法，并覆写带try前缀的方法
* 对于带try前缀的方法，约定通过返回值告知调用者（一般是AQS）获取或释放操作的结果

>本文虽短，且非常重要，是后文分析ReentrantLock等的基础。同时，又是一个高效并发的经典设计案例。
