---
title: 源码|并发一枝花之BlockingQueue
tags:
  - Java
  - 并发
  - 面试
  - 原创
reward: true
date: 2017-10-18 12:33:55
---

今天来介绍Java并发编程中最受欢迎的同步类——堪称并发一枝花之BlockingQueue。

<!--more-->

>JDK版本：oracle java 1.8.0_102

继续阅读之前，需**确保你对锁和条件队列的使用方法烂熟于心，特别是条件队列**，否则你可能无法理解以下源码的精妙之处，甚至基本的正确性。本篇暂不涉及此部分内容，需读者自行准备。

# 接口定义

BlockingQueue继承自Queue，增加了阻塞的入队、出队等特性：

```java
public interface BlockingQueue<E> extends Queue<E> {
  boolean add(E e);

  void put(E e) throws InterruptedException;

  // can extends from Queue. i don't know why overriding here
  boolean offer(E e);

  boolean offer(E e, long timeout, TimeUnit unit)
      throws InterruptedException;

  E take() throws InterruptedException;

  // extends from Queue
  // E poll();

  E poll(long timeout, TimeUnit unit)
      throws InterruptedException;

  int remainingCapacity();

  boolean remove(Object o);

  public boolean contains(Object o);

  int drainTo(Collection<? super E> c);

  int drainTo(Collection<? super E> c, int maxElements);
}
```

>为了方便讲解，我调整了部分方法的顺序，还增加了注释辅助说明。

需要关注的是两对方法:

* **阻塞方法BlockingQueue#put()和BlockingQueue#take()**：如果入队（或出队，下同）失败（如希望入队但队列满，下同），则等待，一直到满足入队条件，入队成功。
* **非阻塞方法BlockingQueue#offer()和BlockingQueue#poll()，及它们的超时版本**：非超时版本是瞬时动作，如果入队当前入队失败，则立刻返回失败；超时版本可在此基础上阻塞一段时间，相当于限时的BlockingQueue#put()和BlockingQueue#take()。

# 实现类

BlockingQueue有很多实现类。根据github的code results排名，最常用的是LinkedBlockingQueue(253k)和ArrayBlockingQueue（95k）。LinkedBlockingQueue的性能在大部分情况下优于ArrayBlockingQueue，本文主要介绍LinkedBlockingQueue，文末会简要提及二者的对比。

## LinkedBlockingQueue

### 阻塞方法put()和take()

两个阻塞方法相对简单，有助于理解LinkedBlockingQueue的核心思想：**在队头和队尾各持有一把锁，入队和出队之间不存在竞争**。

>前面在[Java实现生产者-消费者模型](/2017/10/08/Java实现生产者-消费者模型/)中循序渐进的引出了BlockingQueue#put()和BlockingQueue#take()的实现，可以先去复习一下，了解为什么LinkedBlockingQueue要如此设计。以下是更细致的讲解。

#### 阻塞的入队操作put()

在队尾入队。putLock和notFull配合完成同步。

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
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
```

现在触发一个入队操作，分情况讨论。

##### case1：入队前，队列非空非满（长度大于等于2）

入队前需得到锁putLock。检查队列非满，无需等待条件notFull，直接入队。入队后，检查队列非满（精确说是入队前“将满”，但不影响理解），随机通知一个生产者条件notFull满足。最后，检查入队前队列非空，则无需通知条件notEmpty。

注意点：

* **入队前队列非空非满（长度大于等于2），则head和tail指向的节点不同，入队与出队操作不会同时更新同一节点也就不存在竞争**。因此，分别用两个锁同步入队、出队操作才能是线程安全的。进一步的，由于入队已经由锁putLock保护，则enqueue内部实现不需要加锁。
* 条件notFull可以只随机通知一个等待该条件的生产者线程（使用signal()而不是signalAll()）。即`“单次通知”`，目的是减少无效竞争。**但这不会产生“信号劫持”的问题，因为只有生产者在等待该条件**。
* 条件通知方法singal()是近乎“幂等”的：如果有线程在等待该条件，则随机选择一个线程通知；如果没有线程等待，则什么都不做，不会造成什么恶劣影响。

##### case2：入队前，队列满

入队前需得到锁putLock。检查队列满，则等待条件notFull。*条件notFull可能由出队成功触发（必要的），也可能由入队成功触发（也是必要的，避免“信号不足”的问题）*。条件notFull满足后，入队。入队后，假设检查队列满（队列非满的情况同case1），则无需通知条件notFull。最后，检查入队前队列非空，则无需通知条件notEmpty。

注意点：

* **“信号不足”问题**：假设队列满时，存在_3个生产者P1-P3（多于一个就可以）同时阻塞在10行_；如果此时_5个消费者C1-C5（多于一个就可以）快速、连续的出队，但最后只会有一个信号发出_（19-20行在take()中的对偶逻辑，只会在队列之前消费前队列满的情况发出信号）；_一个信号只能唤醒一个生产者P1_，但明显此时队列缺少了5个元素，该逻辑_不足以唤醒P2、P3_。因此，**14-15行“入队完成时的通知”是必要的，保证了只要队列非满，每次入队后都能唤醒1个阻塞的生产者，来等待锁释放后竞争锁**。即，_P1完成入队后，如果检查到队列非满，会随机唤醒一个生产者P2，让P2在P1释放锁putLock后竞争锁，继续入队_，P3同理。相比于signalAll()唤醒所有生产者，这种解决方案使得同一时间最多只有一个生产者在清醒的竞争锁，性能提升非常明显。

补充signalNotEmpty()、signalNotFull()的实现：

```java
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
}
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal();
    } finally {
        putLock.unlock();
    }
}
```

##### case3：入队前，队列空

入队前需得到锁putLock。检查队列空，则无需等待条件notFull，直接入队。入队后，如果队列非满，则同case1；如果队列满，则同case2。最后，假设检查入队前队列空（队列非空的情况同case1），则随机通知一个消费者条件notEmpty满足。

注意点：

* **只有入队前队列空的情况下，才需要通知条件notEmpty满足**。即`“条件通知”`，是一种减少无效通知的措施。因为如果队列非空，则出队操作不会阻塞在条件notEmpty上。另一方面，虽然已经有生产者完成了入队，但可能有消费者在生产者释放锁putLock后、通知条件notEmpty满足前，使队列变空；不过这没有影响，take()方法的while循环能够在线程竞争到锁之后再次确认。
* 通过入队和出队前检查队列长度（while+await），隐含保证了队列空时只允许入队操作，不存在竞争队列。

##### case4：入队前，队列长度为1

case4是一个特殊情况，分析方法类似于case1，但可能入队与出队之间存在竞争，我们稍后分析。

#### 阻塞的出队操作take()

在队头入队。takeLock和notEmpty配合完成同步。

```java
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
```

依旧是四种case，put()和take()是对偶的，很容易分析，不赘述。

#### “case4 队列长度为1”时的特殊情况

队列长度为1时，到底入队和出队之间存在竞争吗？这取决于LinkedBlockingQueue的底层数据结构。

最简单的是使用朴素链表，可以自己实现，也可以使用JDK提供的非线程安全集合类，如LinkedList等。但是，队列长度为1时，朴素链表中的head、tail指向同一个节点，从而入队、出队更新同一个节点时存在竞争。

>朴素链表：一个节点保存一个元素，不加任何控制和trick。典型如LinkedList。

**增加dummy node可解决该问题**（或者叫哨兵节点什么的）。定义Node(item, next)，描述如下：

* 初始化链表时，创建dummy node：
    * dummy = new Node(null, null)
    * head = dummy.next // head 为 null <=> 队列空
    * tail = dummy // tail.item 为 null <=> 队列空
* 在队尾入队时，tail后移：
    * tail.next = new Node(newItem, null)
    * tail = tail.next
* 在队头出队时，dummy后移，同步更新head：
    * oldItem = head.item
    * dummy = dummy.next
    * dummy.item = null
    * head = dummy.next
    * return oldItem

在新的数据结构中，**更新操作发生在dummy和tail上**，head仅仅作为示意存在，跟随dummy节点更新。队列长度为1时，虽然head、tail仍指向同一个节点，但**dummy、tail指向不同的节点，从而更新dummy和tail时不存在竞争**。

源码中的head即为`dummy`，first即为`head`：

```java
...
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);
}
...
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
    last = last.next = node;
}
...
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
```

#### enqueue和count自增的先后顺序

以put()为例，**count自增一定要晚于enqueue执行**，否则take()方法的while循环检查会失效。

用一个最简单的场景来分析，只有一个生产者线程T1，一个消费者线程T2。

##### 如果先count自增再enqueue

假设目前队列长度0，则事件发生顺序：

1. T1线程：count 自增
2. T2线程：while 检查 count > 0，无需等待条件 notEmpty
3. T2线程：dequeue 执行
4. T1线程：enqueue 执行

很明显，在事件1发生后事件4发生前，虽然count>0，但队列中实际是没有元素的。因此，事件3 dequeue会执行失败（预计抛出NullPointerException）。事件4也就不会发生了。

##### 如果先enqueue再count自增

如果先enqueue再count自增，就不会存在该问题。

仍假设目前队列长度0，则事件发生顺序：

1. T1线程：enqueue 执行
2. T2线程：while 检查 count == 0，等待条件 notEmpty
3. T1线程：count 自增
4. T1线程：通知条件notFull满足
5. T1线程：通知条件notEmpty满足
6. T2线程：收到条件notEmpty
7. T2线程：while 检查 count > 0，无需等待条件 notEmpty
8. T2线程：dequeue 执行

换个方法，用状态机来描述：

* 事件E1发生前，队列处于`状态S1`
* 事件E1发生，线程T1 增加了一个队列元素，导致队列元素的数量大于count（1>0），队列转换到`状态S2`
* 事件E1发生后、直到事件E3发生前，队列一直处于`状态S2`
* 事件E3发生，线程T1 使count自增，导致队列元素的数量等于count（1=1），队列转换到`状态S1`
* 事件E3发生后、事件E8发生前，队列一直处于`状态S1`

>很多读者可能第一次从状态机的角度来理解并发程序设计，所以猴子选择先写出状态迁移序列，如果能理解上述序列，我们再进行进一步的抽象。实际的状态机定义比下面要严谨的多，不过这里的描述已经足够了。

现在补充定义如下，不考虑入队和出队的区别：

* 队列元素的数量等于count的状态定义为`状态S1`
* 队列元素的数量大于count的状态定义为`状态S2`
* enqueue操作定义为状态转换S1->S2
* count自增操作定义为状态转换S2->S1

**LinkedBlockingQueue中的同步机制保证了不会有其他线程看到状态S2**，即，S1->S2->S1两个状态转换只能由线程T1连续完成，其他线程无法在中间插入状态转换。

>在猴子的理解中，**并发程序设计的本质是状态机，即维护合法的状态和状态转换**。以上是一个极其简单的场景，用状态机举例子就可以描述；然而，复杂场景需要用状态机做数学证明，这使得用状态机描述并发程序设计不太受欢迎（虽然口头描述也不能算严格证明）。不过，理解实现中的各种代码顺序、猛不丁蹦出的trick，这些只是“知其所以然”；通过简单的例子来掌握其状态机本质，才能让我们了解其如何保证线程安全性，自己也能写出类似的实现，做到“知其然而知其所以然”。后面会继续用状态机分析ConcurrentLinkedQueue的源码，敬请期待。

### 非阻塞方法offer()和poll()

分析了两个阻塞方法put()、take()后，非阻塞方法就简单了。

#### 瞬时版

以offer为例，poll()同理。假设此时队列非空。

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    final AtomicInteger count = this.count;
    if (count.get() == capacity)
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        if (count.get() < capacity) {
            enqueue(node);
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0;
}
```

##### case1：入队前，队列非满

入队前需得到锁putLock。检查队列非满（隐含表明“无需等待条件notFull”），直接入队。入队后，检查队列非满，随机通知一个生产者（包括使用put()方法的生产者，下同）条件notFull满足。最后，检查入队前队列非空，则无需通知条件notEmpty。

可以看到，瞬时版offer()在队列非满时的行为与put()相同。

##### case2：入队前，队列满

入队前需得到锁putLock。检查队列满，直接退出try-block。后同case1。

队列满时，offer()与put()的区别就显现出来了。put()通过while循环阻塞，一直等到条件notFull得到满足；而offer()却直接返回。

>一个小point：
>
>c在申请锁putLock前被赋值为-1。接下来，如果入队成功，会执行`c = count.getAndIncrement();`一句，则释放锁后，c的值将大于等于0。于是，这里直接用c是否大于等于0来判断是否入队成功。**这种实现牺牲了可读性，只换来了无足轻重的性能或代码量的优化。自己在开发时，不要编写这种代码**。

#### 超时版

同上，以offer()为例。假设此时队列非空。

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    if (e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly();
    try {
        while (count.get() == capacity) {
            if (nanos <= 0)
                return false;
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(new Node<E>(e));
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}
```

该方法同put()很像，12-13行判断nanos超时的情况（吞掉了timeout参数非法的异常情况），所以区别只有14行：将阻塞的`notFull.await()`换成非阻塞的超时版`notFull.awaitNanos(nanos)`。

awaitNanos()的实现有点意思，这里不表。其实现类中的Javadoc描述非常干练：“Block until signalled, interrupted, or timed out.”，返回值为剩余时间。剩余时间小于等于参数nanos，表示：

1. 条件notFull满足（剩余时间大于0）
2. 等待的总时长已超过timeout（剩余时间小于等于0）

nanos首先被初始化为timeout；接下来，消费者线程可能阻塞、收到信号多次，每次收到信号被唤醒，返回的剩余时间都大于0并小于等于参数nanos，再用剩余时间作为下次等待的参数nanos，直到剩余时间小于等于0。以此实现总时长不超过timeout的超时检测。

其他同put()方法。

>12-13行判断nanos参数非法后，直接返回了false。实现有问题，有可能违反接口声明。
>
>根据Javadoc的返回值声明，返回值true表示入队成功，false表示入队失败。但如果传进来的timeout是一个负数，那么5行初始化的nanos也将是一个负数；进而一进入while循环，就在13行返回了false。然而，这是一种参数非法的情况，返回false让人误以为参数正常，只是入队失败。这违反了接口声明，并且非常难以发现。
>
>应该在函数头部就将参数非法的情况检查出来，相应抛出IllegalArgumentException。

## LinkedBlockingQueue与ArrayBlockingQueue的区别

github上LinkedBlockingQueue和ArrayBlockingQueue的使用频率都很高。大部分情况下都可以也建议使用LinkedBlockingQueue，但清楚二者的异同点，方能对症下药，在针对不同的优化场景选择最合适的方案。

相同点：

* 支持有界

不同点

* LinkedBlockingQueue底层用链表实现：ArrayBlockingQueue底层用数组实现
* LinkedBlockingQueue支持不指定容量的无界队列（长度最大值Integer.MAX_VALUE）；ArrayBlockingQueue必须指定容量，无法扩容
* LinkedBlockingQueue支持懒加载：ArrayBlockingQueue不支持
* ArrayBlockingQueue入队时不生成额外对象：LinkedBlockingQueue需生成Node对象，消耗时间，且GC压力大
* LinkedBlockingQueue的入队和出队分别用两把锁保护，无竞争，二者不会互相影响；ArrayBlockingQueue的入队和出队共用一把锁，入队和出队存在竞争，一方速度高时另一方速度会变低。不考虑分配对象、GC等因素的话，ArrayBlockingQueue并发性能要低于LinkedBlockingQueue

可以看到，LinkedBlockingQueue整体上是优于ArrayBlockingQueue的。所以，除非某些特殊原因，否则应优先使用LinkedBlockingQueue。

>可能不全，欢迎评论，随时增改。

# 总结

没有。
