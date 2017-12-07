---
title: 条件队列大法好：使用wait、notify和notifyAll的正确姿势
tags:
  - Java
  - 并发
  - 猿格
  - 原创
reward: true
date: 2017-11-29 14:38:04
---

前面介绍wait和notify的基本语义，参考[条件队列大法好：wait和notify的基本语义](/2017/11/15/条件队列大法好：wait和notify的基本语义/)。这篇讲讲使用wait、notify、notifyAll的正确姿势。

<!--more-->

>一定要先看语义，保证自己掌握了基本语义，再来学习如何使用。

# 基本原理

## 状态依赖的类

状态依赖的类：在状态依赖的类中，存在着某些操作，它们拥有基于状态的前提条件。也就是说，**只有该状态满足某种前提条件时，操作才会继续执行**。

例如，要想从空队列中取得元素，必须等待队列的状态变为“非空”；在这个前提条件得到满足之前，获取元素的操作将保持阻塞。

>如果是头一次了解状态依赖类的概念，很容易将状态依赖类与并发容器混淆。实际上，二者是不对等的概念：
>
>* 并发容器的关键词是“**容器**”，其_提供了不同的并发特征（包括性能、安全、活跃性等方面），用户大多数时候可以直接使用这些容器_。
>* 状态依赖的类的关键词是“**依赖**”，其_提供的是状态同步的基本逻辑，往往用于维护并发程序的状态，例如构建并发容器等_，也可以直接由用户使用。
>

## 可阻塞的状态依赖操作

<!--可阻塞的状态依赖操作的本质-->

状态依赖类的核心是状态依赖操作，最常用的是可阻塞的状态依赖操作。

其本质如下：

```java
acquire lock(on object state) // 测试前需要获取锁，以保证测试时条件不变
while (precondition does not hold) { // pre-check防止信号丢失；re-check防止过早唤醒
  release lock // 如果条件尚未满足，就释放锁，允许其他线程修改条件
  wait until precondition might hold, interrupted or timeout expires
  acquire lock // 再次测试前需要获取锁，以保证测试时条件不变
}
do sth // 如果条件已满足，就执行动作
release lock // 最后再释放锁
```

>注释内容可暂时不关注，后面逐项解释。

对应修改状态的操作：

```java
acquire lock(on object state)
do sth, to make precondition might be hold
release lock
```

条件队列的核心行为就是一个可阻塞的状态依赖操作。

在条件队列中，precondition（前置条件）是一个单元的条件谓词，也即条件队列等待的条件（signal/notify）。大部分使用条件队列的场景，本质上**是在基于单元条件谓词构造多元条件谓词的状态依赖类**。

# 正确姿势

## version1：baseline

如果将具体场景中的多元条件谓词称为“条件谓词”，那么，构造出来的仍然是一个可阻塞的状态依赖操作。

>可以认为，条件谓词和条件队列针对的都是同一个“条件”，只不过条件谓词刻画该“条件”的内容，条件队列用于维护状态依赖，即4行的“`wait until`”。

理解了这一点后，基于条件队列的同步将变的非常简单。大体上是使用Java提供的API实现可阻塞的状态依赖操作。

### key point

基本点：

* 在等待线程中获取条件谓词的状态，如果不满足就等待，满足就继续操作
* 在通知线程中修改条件谓词的状态，之后发出通知

加锁：

* 获取、修改条件谓词的状态是互斥的，需要加锁保护
* 满足条件谓词的值后，需要保证操作期间，条件谓词的状态不变，因此，等待线程的加锁范围应扩展为从检查条件之前开始，然后进入等待，最后到操作之后结束
* 同一时间，只能执行一种操作，对应条件谓词的一次状态转换，因此，通知线程的加锁范围应扩展为从操作之前开始，到发出通知之后结束

API相关：

* 在通知线程等待时，通知线程需要释放自己持有的锁，待条件谓词满足时重新竞争锁。因此，我们在“通知-等待”模型中使用的锁必须与条件队列关联——**在Java中，这一语义都由wait()方法完成，因此，不需要用户显示的释放锁和获取锁**。

### 伪码

使用共享对象shared中的内置锁与内置条件队列。

```java
// 等待线程
synchronized (shared) {
  if (precondition does not hold) {
    shared.wait();
  }
  do sth;
}
```

```java
// 通知线程
synchronized (shared) {
  do sth, to make precondition might be hold;
  shared.notify();
}
```

## version2：过早唤醒

Java提供的条件队列（无论是内置条件队列还是显示条件队列）本身不支持多元条件谓词，因此尽管我们试图基于条件队列内置的单元条件谓词构造多元条件谓词的状态依赖类，但实际上二者在语义上无法绑定在一起——这导致了很多问题。

仍旧以内置条件队列为例。它提供了内置单元条件谓词上的“等待”和“通知”的语义，当内置单元条件谓词满足时，等待线程被唤醒，但该线程无法得知是否是多元条件谓词是否也已经满足。不考虑恶意代码，被唤醒通常有以下原因：

* 自己的多元条件谓词得到满足（这是我们最期望的情况）
* 超时（如果你不希望一直等下去的话）
* 被中断
* 与你共用一个条件队列的多元条件谓词得到满足（我们不建议这样做，但内置条件队列经常会遇到这样的情况）
* 如果你恰好使用了一个线程对象s作为条件队列，那么线程死亡的时候，会自动唤醒等待s的线程

所以，**当线程从wait()方法返回时，必须再次检查多元条件谓词是否满足**。改起来很简单：

```java
// 等待线程
synchronized (shared) {
  while (precondition does not hold) {
    shared.wait();
  }
  do sth;
}
```

另一方面，就算这次被唤醒是因为多元条件谓词得到满足，仍然需要再次检查。别忘了，wait()方法完成了“释放锁->等待通知->收到通知->竞争锁->重新获取锁”一系列事件，虽然“收到通知”时多元条件谓词已经得到满足，但从“收到通知”到“重新获取锁”之间，可能有其他线程已经获取了这个锁，并修改了多元条件谓词的状态，使得多元条件谓词再次变得不满足。

以上几种情况即为“`过早唤醒`”。

## version3：信号丢失

还有一个很难注意到的问题：*re-check时，使用while-do还是do-while？*

本质上是一个”先检查还是先wait“的问题，发生在等待线程和通知线程启动的过程中。假设使用do-while：如果通知线程先发出通知，等待线程再进入等待，那么等待线程将永远不会醒来，也就是“`信号丢失`”。这是因为，条件队列的通知没有“`粘附性`”：如果条件队列收到通知时，没有线程等待，通知就被丢弃了。

要解决信号丢失问题，必须“**先检查再wait**”，使用while-do即可。

## version4：信号劫持

明确了过早唤醒和信号丢失的问题，再来讲信号劫持就容易多了。

信号劫持发生在使用notify()时，notifyAll()不会出现该问题。

假设等待线程T1、T2的条件谓词不同，但共用一个条件队列s。此时，T2的条件谓词得到满足，s收到通知，随机从等待在s上的T1、T2中选择了T1。T1的条件谓词还未满足，经过re-check后再次进入了阻塞状态；而条件谓词已经满足的T2却没有被唤醒。由于T1的过早唤醒，使得T2的信号丢失了，我们就说在T2上发生了信号劫持。

**将通知线程代码中的notify()替换为notifyAll()可以解决信号劫持的问题**：

```java
// 通知线程
synchronized (shared) {
  do sth, to make precondition might be hold;
  shared.notifyAll();
}
```

不过，notifyAll()的副作用非常大：一次性唤醒等待在条件队列上的所有线程，除了最终竞争到锁的线程，其他线程都相当于无效竞争。事实上，使用notify()也可以，只需要保证每次都能叫醒正确的等待线程。方法很简单：

* **一个条件队列只与一个多元条件谓词绑定**，即“`单进单出`”。

如果使用内置条件队列，由于一个内置锁只关联了一个内置条件队列，单进单出的条件将很难满足（如队列非空与队列非满）。显式锁（如ReentrantLock）提供了Lock#newCondition()方法，能在一个显式锁上创建多个显示条件队列，能保证满足该条件。

总之，信号劫持问题需要在设计状态依赖类的时候解决。如果可以避免信号劫持，还是要使用notify()：

```java
// 通知线程
synchronized (shared) {
  do sth, to make precondition might be hold;
  shared.notify();
}
```

## final version

大体框架记住后，使用条件队列的正确姿势可以精简为以下几个要点：

* 全程加锁
* while-do 等待
* 要想使用notify，必须保证单进单出

最后给一个之前手撸的生产者消费者模型，明确使用wait、notify、notifyAll的正确姿势，详细参考[Java实现生产者-消费者模型](/2017/10/08/Java实现生产者-消费者模型/)。

该例中，生产者与消费者互为等待线程与通知线程；两个条件谓词非空`buffer.size() > 0`与非满`buffer.size() < cap`共用同一个条件队列`BUFFER_LOCK`，需要使用notifyAll避免信号劫持。简化如下：

```java
public class WaitNotifyModel implements Model {
  private final Object BUFFER_LOCK = new Object();
  private final Queue<Task> buffer = new LinkedList<>();
...
  private class ConsumerImpl extends AbstractConsumer implements Consumer, Runnable {
    @Override
    public void consume() throws InterruptedException {
      synchronized (BUFFER_LOCK) {
        while (buffer.size() == 0) {
          BUFFER_LOCK.wait();
        }
        Task task = buffer.poll();
        assert task != null;
        // 固定时间范围的消费，模拟相对稳定的服务器处理过程
        Thread.sleep(500 + (long) (Math.random() * 500));
        System.out.println("consume: " + task.no);
        BUFFER_LOCK.notifyAll();
      }
    }
  }

  private class ProducerImpl extends AbstractProducer implements Producer, Runnable {
    @Override
    public void produce() throws InterruptedException {
      // 不定期生产，模拟随机的用户请求
      Thread.sleep((long) (Math.random() * 1000));
      synchronized (BUFFER_LOCK) {
        while (buffer.size() == cap) {
          BUFFER_LOCK.wait();
        }
        Task task = new Task(increTaskNo.getAndIncrement());
        buffer.offer(task);
        System.out.println("produce: " + task.no);
        BUFFER_LOCK.notifyAll();
      }
    }
  }
...
}
```

建议感兴趣的读者继续阅读[源码|并发一枝花之BlockingQueue](/2017/10/18/源码%7C并发一枝花之BlockingQueue/)，从LinkedBlockingQueue的实现中，学习如何保证“一个条件队列只与一个多元条件谓词绑定”以避免信号劫持，还能了解到"`单次通知`"、"`条件通知`"   等常见优化手段。

# 总结

条件队列的使用是并发面试中的一个好考点。猴子第一次遇到时一脸懵逼，叽里咕噜也没有答上来，现在写文章时才发现自己根本没有理解。如果本文有哪里说错了，希望您能通过简书或邮箱联系我，提前致谢。

>挖坑系列——以后讲一下wait、notify、notifyAll的实现机制。
