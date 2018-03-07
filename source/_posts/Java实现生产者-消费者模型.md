---
title: Java实现生产者-消费者模型
tags:
  - Java
  - 并发
  - 面试
  - 原创
reward: true
date: 2017-10-08 17:36:55
---

考查Java的并发编程时，手写“生产者-消费者模型”是一个经典问题。有如下几个考点：

* 对Java并发模型的理解
* 对Java并发编程接口的熟练程度
* bug free
* coding style

<!--more-->

>JDK版本：oracle java 1.8.0_102

本文主要归纳了4种写法，阅读后，最好在白板上练习几遍，检查自己是否掌握。这4种写法或者编程接口不同，或者并发粒度不同，但本质是相同的——都是在使用或实现BlockingQueue。

# 生产者-消费者模型

网上有很多生产者-消费者模型的定义和实现。本文研究最常用的**有界**生产者-消费者模型，简单概括如下：

* 生产者持续生产，直到缓冲区满，阻塞；缓冲区不满后，继续生产
* 消费者持续消费，直到缓冲区空，阻塞；缓冲区不空后，继续消费
* 生产者可以有多个，消费者也可以有多个

可通过如下条件验证模型实现的正确性：

* 同一产品的消费行为一定发生在生产行为之后
* 任意时刻，缓冲区大小不小于0，不大于限制容量

该模型的应用和变种非常多，不赘述。

# 几种写法

## 准备

>面试时可语言说明以下准备代码。关键部分需要实现，如AbstractConsumer。

下面会涉及多种生产者-消费者模型的实现，可以先抽象出关键的接口，并实现一些抽象类：

```java
public interface Consumer {
  void consume() throws InterruptedException;
}
```

```java
public interface Producer {
  void produce() throws InterruptedException;
}
```

```java
abstract class AbstractConsumer implements Consumer, Runnable {
  @Override
  public void run() {
    while (true) {
      try {
        consume();
      } catch (InterruptedException e) {
        e.printStackTrace();
        break;
      }
    }
  }
}
```

```java
abstract class AbstractProducer implements Producer, Runnable {
  @Override
  public void run() {
    while (true) {
      try {
        produce();
      } catch (InterruptedException e) {
        e.printStackTrace();
        break;
      }
    }
  }
}
```

不同的模型实现中，生产者、消费者的具体实现也不同，所以需要为模型定义抽象工厂方法：

```java
public interface Model {
  Runnable newRunnableConsumer();

  Runnable newRunnableProducer();
}
```

我们将Task作为生产和消费的单位：

```java
public class Task {
  public int no;

  public Task(int no) {
    this.no = no;
  }
}
```

>如果需求还不明确（这符合大部分工程工作的实际情况），建议边实现边抽象，**不要“面向未来编程”**。

## 实现一：BlockingQueue

BlockingQueue的写法最简单。核心思想是，**把并发和容量控制封装在缓冲区中**。而BlockingQueue的性质天生满足这个要求。

```java
public class BlockingQueueModel implements Model {
  private final BlockingQueue<Task> queue;

  private final AtomicInteger increTaskNo = new AtomicInteger(0);

  public BlockingQueueModel(int cap) {
    // LinkedBlockingQueue 的队列不 init，入队时检查容量；ArrayBlockingQueue 在创建时 init
    this.queue = new LinkedBlockingQueue<>(cap);
  }

  @Override
  public Runnable newRunnableConsumer() {
    return new ConsumerImpl();
  }

  @Override
  public Runnable newRunnableProducer() {
    return new ProducerImpl();
  }

  private class ConsumerImpl extends AbstractConsumer implements Consumer, Runnable {
    @Override
    public void consume() throws InterruptedException {
      Task task = queue.take();
      // 固定时间范围的消费，模拟相对稳定的服务器处理过程
      Thread.sleep(500 + (long) (Math.random() * 500));
      System.out.println("consume: " + task.no);
    }
  }

  private class ProducerImpl extends AbstractProducer implements Producer, Runnable {
    @Override
    public void produce() throws InterruptedException {
      // 不定期生产，模拟随机的用户请求
      Thread.sleep((long) (Math.random() * 1000));
      Task task = new Task(increTaskNo.getAndIncrement());
      System.out.println("produce: " + task.no);
      queue.put(task);
    }
  }

  public static void main(String[] args) {
    Model model = new BlockingQueueModel(3);
    for (int i = 0; i < 2; i++) {
      new Thread(model.newRunnableConsumer()).start();
    }
    for (int i = 0; i < 5; i++) {
      new Thread(model.newRunnableProducer()).start();
    }
  }
}
```

截取前面的一部分输出：

```
produce: 0
produce: 4
produce: 2
produce: 3
produce: 5
consume: 0
produce: 1
consume: 4
produce: 7
consume: 2
produce: 8
consume: 3
produce: 6
consume: 5
produce: 9
consume: 1
produce: 10
consume: 7
```

由于操作“出队/入队+日志输出”不是原子的，所以上述日志的绝对顺序与实际的出队/入队顺序有出入，但对于同一个任务号`task.no`，其consume日志一定出现在其produce日志之后，即：同一任务的消费行为一定发生在生产行为之后。缓冲区的容量留给读者验证。符合两个验证条件。

BlockingQueue写法的核心只有两行代码，并发和容量控制都封装在了BlockingQueue中，正确性由BlockingQueue保证。面试中首选该写法，自然美观简单。

>勘误：
>
>在简书回复一个读者的时候，顺道发现了这个问题：生产日志应放在入队操作之前，否则同一个task的生产日志可能出现在消费日志之后。
>
>```java
// 旧的错误代码
queue.put(task);
System.out.println("produce: " + task.no);
```

>```java
// 正确代码
System.out.println("produce: " + task.no);
queue.put(task);
```

>具体来说，生产日志应放在入队操作之前，消费日志应放在出队操作之后，以保障：
>
>* 消费线程中queue.take()返回之后，对应生产线程（生产该task的线程）中queue.put()及之前的行为，对于消费线程来说都是可见的。
>
>想想为什么呢？因为我们需要借助“queue.put()与queue.take()的偏序关系”。其他实现方案分别借助了条件队列、锁的偏序关系，不存在该问题。要解释这个问题，需要读者明白可见性和Happens-Before的概念，篇幅所限，暂时不多解释。
>
>PS：旧代码没出现这个问题，是因为消费者打印消费日志之前，sleep了500+ms，而恰巧竞争不激烈，这个时间一般足以让“滞后”生产日志打印完成（但不保证）。
>
>---
>
>顺道说明一下，猴子现在主要在个人博客、简书、掘金和CSDN上发文章，搜索“猴子007”或“程序猿说你好”都能找到。但个人精力有限，部分勘误难免忘记同步到某些地方（甚至连新文章都不同步了T_T），只能保证个人博客是最新的，还望理解。
>
>写文章不是为了出名，一方面希望整理自己的学习成果，一方面希望有更多人能帮助猴子纠正学习过程中的错误。如果能认识一些志同道合的朋友，一起提高就更好了。所以希望各位转载的时候，一定带着猴子个人博客末尾的转载声明。需要联系猴子的话，简书或邮件都可以。
>
>文章水平不高，就不奢求有人能打赏鼓励我这泼猴了T_T

## 实现二：wait && notify

如果不能将并发与容量控制都封装在缓冲区中，就只能由消费者与生产者完成。最简单的方案是使用朴素的`wait && notify`机制。

```java
public class WaitNotifyModel implements Model {
  private final Object BUFFER_LOCK = new Object();
  private final Queue<Task> buffer = new LinkedList<>();
  private final int cap;

  private final AtomicInteger increTaskNo = new AtomicInteger(0);

  public WaitNotifyModel(int cap) {
    this.cap = cap;
  }

  @Override
  public Runnable newRunnableConsumer() {
    return new ConsumerImpl();
  }

  @Override
  public Runnable newRunnableProducer() {
    return new ProducerImpl();
  }

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

  public static void main(String[] args) {
    Model model = new WaitNotifyModel(3);
    for (int i = 0; i < 2; i++) {
      new Thread(model.newRunnableConsumer()).start();
    }
    for (int i = 0; i < 5; i++) {
      new Thread(model.newRunnableProducer()).start();
    }
  }
}
```

验证方法同上。

朴素的`wait && notify`机制不那么灵活，但足够简单。synchronized、wait、notifyAll的用法可参考[【Java并发编程】之十：使用wait/notify/notifyAll实现线程间通信的几点重要说明](http://blog.csdn.net/ns_code/article/details/17225469)，**着重理解唤醒与锁竞争的区别**，猴子以后也会撰文介绍。

## 实现三：简单的Lock && Condition

我们要保证理解`wait && notify`机制。实现时可以使用Object类提供的wait()方法与notifyAll()方法，但更推荐的方式是使用java.util.concurrent包提供的`Lock && Condition`。

```java
public class LockConditionModel1 implements Model {
  private final Lock BUFFER_LOCK = new ReentrantLock();
  private final Condition BUFFER_COND = BUFFER_LOCK.newCondition();
  private final Queue<Task> buffer = new LinkedList<>();
  private final int cap;

  private final AtomicInteger increTaskNo = new AtomicInteger(0);

  public LockConditionModel1(int cap) {
    this.cap = cap;
  }

  @Override
  public Runnable newRunnableConsumer() {
    return new ConsumerImpl();
  }

  @Override
  public Runnable newRunnableProducer() {
    return new ProducerImpl();
  }

  private class ConsumerImpl extends AbstractConsumer implements Consumer, Runnable {
    @Override
    public void consume() throws InterruptedException {
      BUFFER_LOCK.lockInterruptibly();
      try {
        while (buffer.size() == 0) {
          BUFFER_COND.await();
        }
        Task task = buffer.poll();
        assert task != null;
        // 固定时间范围的消费，模拟相对稳定的服务器处理过程
        Thread.sleep(500 + (long) (Math.random() * 500));
        System.out.println("consume: " + task.no);
        BUFFER_COND.signalAll();
      } finally {
        BUFFER_LOCK.unlock();
      }
    }
  }

  private class ProducerImpl extends AbstractProducer implements Producer, Runnable {
    @Override
    public void produce() throws InterruptedException {
      // 不定期生产，模拟随机的用户请求
      Thread.sleep((long) (Math.random() * 1000));
      BUFFER_LOCK.lockInterruptibly();
      try {
        while (buffer.size() == cap) {
          BUFFER_COND.await();
        }
        Task task = new Task(increTaskNo.getAndIncrement());
        buffer.offer(task);
        System.out.println("produce: " + task.no);
        BUFFER_COND.signalAll();
      } finally {
        BUFFER_LOCK.unlock();
      }
    }
  }

  public static void main(String[] args) {
    Model model = new LockConditionModel1(3);
    for (int i = 0; i < 2; i++) {
      new Thread(model.newRunnableConsumer()).start();
    }
    for (int i = 0; i < 5; i++) {
      new Thread(model.newRunnableProducer()).start();
    }
  }
}
```

该写法的思路与实现二的思路完全相同，仅仅将锁与条件变量换成了Lock和Condition。

## 实现四：更高并发性能的Lock && Condition

现在，如果做一些实验，你会发现，**实现一的并发性能高于实现二、三**。暂且不关心BlockingQueue的具体实现，来分析看如何优化实现三（与实现二的思路相同，性能相当）的性能。

### 分析实现三的瓶颈

>最好的查证方法是记录方法执行时间，这样可以直接定位到真正的瓶颈。但此问题较简单，我们直接用“瞪眼法”分析。

实现三的并发瓶颈很明显，因为在锁 `BUFFER_LOCK` 看来，任何消费者线程与生产者线程都是一样的。换句话说，同一时刻，最多只允许有一个线程（生产者或消费者，二选一）操作缓冲区 buffer。

而实际上，如果缓冲区是一个队列的话，“生产者将产品入队”与“消费者将产品出队”两个操作之间没有同步关系，可以在队首出队的同时，在队尾入队。**理想性能可提升至实现三的两倍**。

### 去掉这个瓶颈

那么思路就简单了：需要两个锁 `CONSUME_LOCK`与`PRODUCE_LOCK`，`CONSUME_LOCK`控制消费者线程并发出队，`PRODUCE_LOCK`控制生产者线程并发入队；相应需要两个条件变量`NOT_EMPTY`与`NOT_FULL`，`NOT_EMPTY`负责控制消费者线程的状态（阻塞、运行），`NOT_FULL`负责控制生产者线程的状态（阻塞、运行）。以此让优化消费者与消费者（或生产者与生产者）之间是串行的；消费者与生产者之间是并行的。

```java
public class LockConditionModel2 implements Model {
  private final Lock CONSUME_LOCK = new ReentrantLock();
  private final Condition NOT_EMPTY = CONSUME_LOCK.newCondition();
  private final Lock PRODUCE_LOCK = new ReentrantLock();
  private final Condition NOT_FULL = PRODUCE_LOCK.newCondition();

  private final Buffer<Task> buffer = new Buffer<>();
  private AtomicInteger bufLen = new AtomicInteger(0);

  private final int cap;

  private final AtomicInteger increTaskNo = new AtomicInteger(0);

  public LockConditionModel2(int cap) {
    this.cap = cap;
  }

  @Override
  public Runnable newRunnableConsumer() {
    return new ConsumerImpl();
  }

  @Override
  public Runnable newRunnableProducer() {
    return new ProducerImpl();
  }

  private class ConsumerImpl extends AbstractConsumer implements Consumer, Runnable {
    @Override
    public void consume() throws InterruptedException {
      int newBufSize = -1;

      CONSUME_LOCK.lockInterruptibly();
      try {
        while (bufLen.get() == 0) {
          System.out.println("buffer is empty...");
          NOT_EMPTY.await();
        }
        Task task = buffer.poll();
        newBufSize = bufLen.decrementAndGet();
        if (newBufSize > 0) {
          NOT_EMPTY.signalAll();
        }
      } finally {
        CONSUME_LOCK.unlock();
      }

      if (newBufSize < cap) {
        PRODUCE_LOCK.lockInterruptibly();
        try {
          NOT_FULL.signalAll();
        } finally {
          PRODUCE_LOCK.unlock();
        }
      }
      
      assert task != null;
      // 固定时间范围的消费，模拟相对稳定的服务器处理过程
      Thread.sleep(500 + (long) (Math.random() * 500));
      System.out.println("consume: " + task.no);
    }
  }

  private class ProducerImpl extends AbstractProducer implements Producer, Runnable {
    @Override
    public void produce() throws InterruptedException {
      // 不定期生产，模拟随机的用户请求
      Thread.sleep((long) (Math.random() * 1000));

      int newBufSize = -1;

      PRODUCE_LOCK.lockInterruptibly();
      try {
        while (bufLen.get() == cap) {
          System.out.println("buffer is full...");
          NOT_FULL.await();
        }
        Task task = new Task(increTaskNo.getAndIncrement());
        buffer.offer(task);
        newBufSize = bufLen.incrementAndGet();
        System.out.println("produce: " + task.no);
        if (newBufSize < cap) {
          NOT_FULL.signalAll();
        }
      } finally {
        PRODUCE_LOCK.unlock();
      }

      if (newBufSize > 0) {
        CONSUME_LOCK.lockInterruptibly();
        try {
          NOT_EMPTY.signalAll();
        } finally {
          CONSUME_LOCK.unlock();
        }
      }
    }
  }

  private static class Buffer<E> {
    private Node head;
    private Node tail;

    Buffer() {
      // dummy node
      head = tail = new Node(null);
    }

    public void offer(E e) {
      tail.next = new Node(e);
      tail = tail.next;
    }

    public E poll() {
      head = head.next;
      E e = head.item;
      head.item = null;
      return e;
    }

    private class Node {
      E item;
      Node next;

      Node(E item) {
        this.item = item;
      }
    }
  }

  public static void main(String[] args) {
    Model model = new LockConditionModel2(3);
    for (int i = 0; i < 2; i++) {
      new Thread(model.newRunnableConsumer()).start();
    }
    for (int i = 0; i < 5; i++) {
      new Thread(model.newRunnableProducer()).start();
    }
  }
```

需要注意的是，由于需要同时在UnThreadSafe的缓冲区 buffer 上进行消费与生产，我们不能使用实现二、三中使用的队列了，需要自己实现一个简单的缓冲区 Buffer。Buffer要满足以下条件：

* 在头部出队，尾部入队
* 在poll()方法中只操作head
* 在offer()方法中只操作tail

### 还能进一步优化吗

我们已经优化掉了消费者与生产者之间的瓶颈，还能进一步优化吗？

如果可以，必然是继续优化消费者与消费者（或生产者与生产者）之间的并发性能。然而，消费者与消费者之间必须是串行的，因此，并发模型上已经没有地方可以继续优化了。

>仔细分析下，实现四中的signalAll都可以换成signal。这里为了屏蔽复杂性，回避了这个优化。

不过在具体的业务场景中，一般还能够继续优化。如：

* 并发规模中等，可考虑使用CAS代替重入锁
* 模型上不能优化，但一个消费行为或许可以进一步拆解、优化，从而降低消费的延迟
* 一个队列的并发性能达到了极限，可采用“多个队列”（如分布式消息队列等）

# 4种实现的本质

文章开头说：**这4种写法的本质相同——都是在使用或实现BlockingQueue**。实现一直接使用BlockingQueue，实现四实现了简单的BlockingQueue，而实现二、三则实现了退化版的BlockingQueue（性能降低一半）。

实现一使用的BlockingQueue实现类是LinkedBlockingQueue，给出其源码阅读对照，写的不难：

```java
public class LinkedBlockingQueue<E> extends AbstractQueue<E>
        implements BlockingQueue<E>, java.io.Serializable {
...
/** Lock held by take, poll, etc */
    private final ReentrantLock takeLock = new ReentrantLock();
    /** Wait queue for waiting takes */
    private final Condition notEmpty = takeLock.newCondition();
    /** Lock held by put, offer, etc */
    private final ReentrantLock putLock = new ReentrantLock();
    /** Wait queue for waiting puts */
    private final Condition notFull = putLock.newCondition();
...
    /**
     * Signals a waiting take. Called only from put/offer (which do not
     * otherwise ordinarily lock takeLock.)
     */
    private void signalNotEmpty() {
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
    }

    /**
     * Signals a waiting put. Called only from take/poll.
     */
    private void signalNotFull() {
        final ReentrantLock putLock = this.putLock;
        putLock.lock();
        try {
            notFull.signal();
        } finally {
            putLock.unlock();
        }
    }

    /**
     * Links node at end of queue.
     *
     * @param node the node
     */
    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }

    /**
     * Removes a node from head of queue.
     *
     * @return the node
     */
    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }
...
    /**
     * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
     *
     * @param capacity the capacity of this queue
     * @throws IllegalArgumentException if {@code capacity} is not greater
     *         than zero
     */
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
        last = head = new Node<E>(null);
    }
...
    /**
     * Inserts the specified element at the tail of this queue, waiting if
     * necessary for space to become available.
     *
     * @throws InterruptedException {@inheritDoc}
     * @throws NullPointerException {@inheritDoc}
     */
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node<E>(e);
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
            while (count.get() == capacity) {
                notFull.await();
            }
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
...
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
...
}
```

>还存在非常多的写法，如信号量`Semaphore`，也很常见（本科操作系统教材中的生产者-消费者模型就是用信号量实现的）。不过追究过多了就好像在纠结茴香豆的写法一样，本文不继续探讨。

# 总结

实现一必须掌握，实现四至少要能清楚表述；实现二、三掌握一个即可。
