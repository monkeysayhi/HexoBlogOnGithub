---
title: 源码|并发一枝花之ReentrantLock与AQS（2）：lockInterruptibly
tags:
  - Java
  - 并发
  - 原创
reward: true
date: 2017-12-05 22:42:59
---

上次分析了ReentrantLock#lock()与ReentrantLock#unlock()的实现原理，并初步讨论了AQS的等待队列模型，参考[源码|并发一枝花之ReentrantLock与AQS（1）：lock、unlock](/2017/12/05/源码|并发一枝花之ReentrantLock与AQS（1）：lock、unlock/)。

本文继续分析ReentrantLock#lockInterruptibly()，内容简单，可以跳过。

<!--more-->

>JDK版本：oracle java 1.8.0_102

# 接口声明

```java
public interface Lock {
    ...
    void lock();

    void lockInterruptibly() throws InterruptedException;

    void unlock();
    ...
}
```

Lock#lockInterruptibly()是Lock#lock()的一个衍生品，解锁方法也为Lock#unlock()。因此，我们只需要分析Lock#lockInterruptibly()。

# 实现原理

基本的实现原理与ReentrantLock#lock()相同。

仍以默认的非公平策略为例进行分析。 

## lockInterruptibly

```java
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }
```

回忆ReentrantLock的核心思想：**用state表示“所有者线程已经重复获取该锁的次数”**。

### acquireInterruptibly

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    ...
    public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
    ...
}
```

改写：

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    ...
    public final void acquireInterruptibly(int arg)
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquire(arg)) {
            return;
        }
        doAcquireInterruptibly(arg);
    }
    ...
}
```

首先，通过tryAcquire()尝试获取锁。如果获取成功，则直接返回。如果获取失败，则借助doAcquireInterruptibly实现可中断获取。

NonfairSync#tryAcquire()在ReentrantLock#lock()中分析过了，直接看AQS#doAcquireInterruptibly()。

#### doAcquireInterruptibly

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    ...
    private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
    ...
}
```

同AQS#acquireQueued()基本相同，唯一的区别是对中断信号的处理。

AQS#acquireQueued()被中断后，将中断标志传给外界，外界再调用Thread#interrupt()复现中断；而AQS#doAcquireInterruptibly()则直接抛出InterruptedException。

二者本质上没什么不同。但AQS#doAcquireInterruptibly()显示抛出了InterruptedException，调用者必须处理或继续上抛该异常。

# 总结

可以看到，分析完ReentrantLock#lock()后，其衍生品ReentrantLock#lockInterruptibly()的分析也变得极其简单。ReentrantLock#tryLock()与之相似，不再分析。
