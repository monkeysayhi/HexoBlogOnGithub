---
title: 线程池ThreadPoolExecutor总结
tags:
  - Java
  - 并发
  - 面试
  - 原创
reward: true
date: 2018-11-08 12:22:31
---

之前在[源码|从串行线程封闭到对象池、线程池](/2017/10/26/源码|从串行线程封闭到对象池、线程池/)中挖坑说要精炼一篇短文。本文填坑，总结线程池的种类、应用场景、ThreadPoolExecutor参数含义，最后简单介绍如何估算线程池大小。

<!--more-->

>JDK版本：oracle java 1.8.0_102
>
>不同语言、同一语言不同库的线程池实现有差别，不要拘泥于Java这一种，没事看看work stealing等方式也挺有意思的。

# 三句话拿下线程池

线程池内部是一个生产者-消费者模型：

* 用户是生产者。提交任务（task）相当于生产产品。
* 线程池中的线程（worker）是消费者，执行任务相当于消费产品。

详见[源码|从串行线程封闭到对象池、线程池](/2017/10/26/源码|从串行线程封闭到对象池、线程池/)。

# 线程池的种类

可通过Executors中的`静态工厂方法`创建不同特点的线程池，包括：

* FixedThreadPool：维持固定nThreads个线程的线程池；使用无界的异步阻塞队列LinkedBlockingQueue作为任务队列。
* CachedThreadPool：维持最少0、最多`Integer.MAX_VALUE`个线程的线程池，限制线程可缓存60s，超时销毁；使用无界的同步队列SynchronousQueue作为任务队列。
* SingleThreadExecutor：维持固定1个线程的FixedThreadPool。
* ScheduledThreadPool：维持固定corePoolSize个线程的线程池；使用无界的延迟队列DelayedWorkQueue作为任务队列。
* SingleThreadScheduledExecutor：维持固定1个线程的ScheduledThreadPool。
* WorkStealingPool：并行度为parallelism的ForkJoinPool，暂不讨论。

忽略WorkStealingPool，则其他线程池底层都使用了ThreadPoolExecutor，只是参数不同。

# 应用场景

线程池就像水管，任务是水。

* 如果期望水管出水的速度固定，就使用FixedThreadPool。
* 如果期望水管出水的速度可以在水流大时增大，水流小时变小，就使用CachedThreadPool。
* 如果期望水管出水速度恒定为1，就使用SingleThreadExecutor。
* 如果期望水管延迟出水（延迟可控，或周期性），就使用ScheduledThreadPool。

# ThreadPoolExecutor各参数的意义

* corePoolSize：线程池维持的最小线程数（ScheduledThreadPool的该参数设计有误），懒初始化。
* maximumPoolSize：线程池维持的最大线程数。
* keepAliveTime，unit：如果线程池中的线程数大于corePoolSize小于等于maximumPoolSize，则选择闲置时间超过keepAliveTime（单位为unit）的线程销毁，直到无线程可销毁或线程数等于corePoolSize。
* workQueue：应该被称为taskQueue，维护提交到线程池的task。如果设置为无界队列，则线程数量将维持为corePoolSize；否则，当队列满时，尝试增加线程数直到maximumPoolSize或队列不满。
* threadFactory：任务以Runnable、Callable的形式提交，生产线程时使用threadFactory的工厂方法。默认`Executors.defaultThreadFactory()`。
* handler：如果workQueue是有界的，那么当workQueue满时，使用handler处理无法提交的新任务。默认`Executors.defaultHandler`，workQueue满时抛出RejectedExecutionException。

观察corePoolSize、maximumPoolSize、workQueue三者的关系可知：

* 如果使用有界线程池，则最好搭配有界队列，否则maximumPoolSize参数无效。
* 相对的，CachedThreadPool被设计为一种maximumPoolSize无效的缓冲池，同时，因此，必须使用无界的同步队列，让“入队”直接变成“执行”。

# 线程池大小的估算

## 最简化公式

* CPU 密集型应用：线程池大小设置为 **N + 1**
* IO 密集型应用：线程池大小设置为 **2N**

公式的意义在于**避免陷入极端情况**。其中，计算密集型任务假设“`等待时间/计算时间`”等于0，IO密集型任务假设“`等待时间/计算时间`”等于1。

**为什么要有+1呢？**

这是因为，*就算是计算密集型任务，也可能存在缺页等问题（需要了解虚拟内存和物理内存的分配），产生“隐式”的IO*。多一个额外的线程能确保CPU时钟周期不会被浪费，又不至于增加太多线程调度成本。

## 严格公式：

假设每个线程的“`等待时间 / 计算时间`”大小相等，显然，“`计算时间 / (计算时间 + 等待时间)`”也相等。对1个线程而言，只有计算时间占用了`逻辑CPU`，假设这个线程一直运行在同1个逻辑CPU上，显然，该逻辑CPU的`CPU利用率`即等于“`计算时间 / (计算时间 + 等待时间)`”。

对于多个线程的情况是一样的，则有公式：

```
逻辑CPU数 * CPU利用率 / 线程数 = 计算时间 / (计算时间 + 等待时间)
```

倒腾倒腾，得到：

```
线程数 = (1 + 等待时间/计算时间) * 逻辑CPU数 * CPU利用率
```

* “`1 + 等待时间/计算时间`” 只与任务本身有关。
* 逻辑CPU数可通过`cat /proc/cpuinfo | grep -c processor`得到。
* **标准的CPU利用率**要通过实际监控得到，但在估算线程池大小时，应看做“**期望得到的CPU利用率**”，即**可分配给该任务的CPU比例**。如果只打算分配一半CPU给任务的话，就是0.5。

如果估算得到的线程数比较多，那么还要适当提高可分配的CPU比例，因为线程切换的成本随线程数增加而增加。如果竞争较激烈，则可以适当降低可分配的CPU比例，因为竞争通常也会导致线程阻塞，使CPU空闲。
