---
title: 源码|并发一枝花之ConcurrentLinkedQueue【伪】
tags:
  - Java
  - 并发
  - 面试
  - 原创
reward: true
date: 2017-10-22 01:05:35
---

>首先声明，本文是**伪源码分析**。主要是基于状态机自己实现一个简化的并发队列，有助于读者掌握并发程序设计的核心——状态机；最后对源码实现略有提及。

ConcurrentLinkedQueue不支持阻塞，没有BlockingQueue那么易用；但在中等规模的并发场景下，其性能却比BlockingQueue高不少，而且相当稳定。同时，ConcurrentLinkedQueue是学习CAS的经典案例。根据github的code results排名，ConcurrentLinkedQueue（164k）也十分流行，比我想象中的使用量大多了。非常值得一讲。

<!--more-->

对于状态机和并发程序设计的基本理解，可以参考[源码|并发一枝花之BlockingQueue](/2017/10/18/源码%7C并发一枝花之BlockingQueue/)，建议第一次接触状态机的同学速读参考文章之后，再来阅读此文章。

>JDK版本：oracle java 1.8.0_102

# 准备知识：CAS

>读者可以跳过这部分，后面讲到offer()方法的实现时再回顾。

## 悲观锁与乐观锁

* 悲观锁：假定并发环境是悲观的，如果发生并发冲突，就会破坏一致性，所以要通过独占锁彻底禁止冲突发生。有一个经典比喻，“如果你不锁门，那么捣蛋鬼就回闯入并搞得一团糟”，所以“你只能一次打开门放进一个人，才能时刻盯紧他”。
* 乐观锁：假定并发环境是乐观的，即，虽然会有并发冲突，但冲突可发现且不会造成损害，所以，可以不加任何保护，等发现并发冲突后再决定放弃操作还是重试。可类比的比喻为，“如果你不锁门，那么虽然捣蛋鬼会闯入，但他们一旦打算破坏你就能知道”，所以“你大可以放进所有人，等发现他们想破坏的时候再做决定”。

通常认为乐观锁的性能比悲观所更高，特别是在某些复杂的场景。这主要由于悲观锁在加锁的同时，也会把某些不会造成破坏的操作保护起来；而乐观锁的竞争则只发生在最小的并发冲突处，如果用悲观锁来理解，就是“锁的粒度最小”。但乐观锁的设计往往比较复杂，因此，复杂场景下还是多用悲观锁。

>首先保证正确性，有必要的话，再去追求性能。

## CAS

乐观锁的实现往往需要硬件的支持，多数处理器都都实现了一个CAS指令，实现“Compare And Swap”的语义（这里的swap是“换入”，也就是set），构成了基本的乐观锁。

CAS包含3个操作数：

* 需要读写的内存位置V
* 进行比较的值A
* 拟写入的新值B

当且仅当位置V的值等于A时，CAS才会通过原子方式用新值B来更新位置V的值；否则不会执行任何操作。无论位置V的值是否等于A，都将返回V原有的值。

>一个有意思的事实是，“使用CAS控制并发”与“使用乐观锁”并不等价。CAS只是一种手段，既可以实现乐观锁，也可以实现悲观锁。乐观、悲观只是一种并发控制的策略。下文将分别用CAS实现悲观锁和乐观锁？
我们先不讲JDK提供的实现，用状态机模型来分析一下，看我们能不能自己实现一版。

# 队列的状态机模型

状态机模型与是否需要并发无关，一个类不管是否是线程安全的，其状态机模型从类被实现（此时，所有类行为都是确定的）开始就是确定的。接口是类行为的一个子集，我们从接口出发，逐渐构建出简化版ConcurrentLinkedQueue的状态机模型。

## 队列接口

ConcurrentLinkedQueue实现了Queue接口：

```java
public interface BlockingQueue<E> extends Queue<E> {
  boolean add(E e);

  boolean offer(E e);

  E remove();

  E poll();

  E element();
  
  E peek();
}
```

需要关注的是一对方法：

* offer()：入队，成功返回true，失败返回false。JDK中ConcurrentLinkedQueue实现为无界队列，这里我们也只讨论无界的情况——因此，offer()方法必返回true。
* poll()：出队，有元素返回元素，没有就返回null。

同时，理想的线程安全队列中，入队和出队之间不应该存在竞争，这样入队的状态机模型和出队的状态机模型可以完全解耦，互不影响。

对我们的状态机作出两个假设：

* 假设1：只支持这入队、出队两种行为。
* 假设2：入队、出队之间不存在竞争，即入队模型与出队模型是对偶、独立的两个状态机。

从而，可以**先分析入队，再参照分析出队；然后可尝试去掉假设2，看如何完善我们的实现来保证假设2成立；最后看看真·神Doug Lea如何实现，学习一波**。

## 状态机定义

现在基于假设1和假设2，尝试定义入队模型的状态机。

我们构造一个简化的场景：**存在2个生产者P1、P2，同时触发入队操作**。

### 状态集

如果是单线程环境，入队操作将是这样的：

```java
// 准备
newNode.next = null;
curTail = tail;

// 入队前
assert tail == curTail && tail.next == null; // 状态S1
// 开始入队
tail.next = newNode; // 事件E1
// 入队中
assert tail == curTail && tail.next == newNode; // 状态S2
tail = tail.next; // 事件E2
// 结束入队
// 入队后
assert tail == newNode && tail.next == null; // 状态S3，合并到状态S1
```

该过程涉及对两个域的修改：tail.next、tail。则随着操作的进行，队列会经历2种状态：

* 状态S1：事件E1执行前，tail指向实际的尾节点curTail，tail.next==null。如生产者P1、P2都还没有触发入队时，队列处于状态S1；生产者P1完成入队P2还没触发入队时，队列处于状态S1。
* 状态S2：事件E1执行后、E2执行前，tail指向旧的尾节点curTail，tail.next==newNode。
* ~~状态S3：事件E2执行后，tail指向新的尾节点newNode，tail.next==null。同状态S1，合并。~~

### 状态转换集

两个事件分别对应两个状态转换：

* 状态转换T1：S1->S2，即tail.next = newNode。
* 状态转换T2：S2->S1，即tail = tail.next。

>是不是很熟悉？因为ConcurrentLinkedQueue也是队列，必然同BlockingQueue相似甚至相同。区别在于如何维护这些状态和状态转换。

# 自撸ConcurrentLinkedQueue

依赖CAS，两个状态转换T1、T2都可以实现为原子操作。留给我们的问题是，如何维护合法的状态转换。

## 入队方法offer()

入队过程需要经过两个状态转换，且这两个状态转换必须连续发生。

>不严谨。“连续”并不是必要的，最后分析源码的时候会看到。不过，我们暂时使用强一致性的模型。

### 思路1：让同一个生产者P1连续完成两个状态转换T1、T2，保证P2不会插入进来

LinkedBlockingQueue的思路即是如此。这是一种悲观策略——一次开门只放进来一个生产者，似乎只能像LinkedBlockingQueue那样，用传统的锁putLock实现，实际上，依靠CAS也能实现：

```java
public class ConcurrentLinkedQueue1<E> {
  private volatile Node<E> tail;

  public ConcurrentLinkedQueue1() {
    throw new UnsupportedOperationException("Not implement");
  }

  public boolean offer(E e) {
    Node<E> newNode = new Node<E>(e, new AtomicReference<>(null));
    while (true) {
      Node<E> curTail = tail;
      AtomicReference<Node<E>> curNext = curTail.next;
      // 尝试T1：CAS设置tail.next
      if (curNext.compareAndSet(null, newNode)) {
        // 成功者视为获得独占锁，完成了T1。直接执行T2：设置tail
        tail = curNext.get();
        return true;
      }
      // 失败者自旋等待
    }
  }

  private static class Node<E> {
    private volatile E item;
    private AtomicReference<Node<E>> next;

    public Node(E item, AtomicReference<Node<E>> next) {
      this.item = item;
      this.next = next;
    }
  }
}
```

### 思路2：生产者P1完成状态转换T1后，P2代劳完成状态转换T2

再来分析下T1、T2两个状态转换：

* T1涉及newNode，只能由封闭持有newNode的生产者P1完成
* T2只涉及队列中的信息，任何持有队列的生产者都有能力完成。P1可以，P2也可以

思路1是悲观的，认为T1、T2必须都由P1完成，如果P2插入就会“搞破坏”。而思路2则打开大门，欢迎任何“有能力”的生产者完成T2，是典型的乐观策略。

```java
public class ConcurrentLinkedQueue2<E> {
  private AtomicReference<Node<E>> tail;

  public ConcurrentLinkedQueue2() {
    throw new UnsupportedOperationException("Not implement");
  }

  public boolean offer(E e) {
    Node<E> newNode = new Node<E>(e, new AtomicReference<>(null));
    while (true) {
      Node<E> curTail = tail.get();
      AtomicReference<Node<E>> curNext = curTail.next;
      // 尝试T1：CAS设置tail.next
      if (curNext.compareAndSet(null, newNode)) {
        // 成功者完成了T1，队列处于S2，继续尝试T2：CAS设置tail
        tail.compareAndSet(curTail, curNext.get());
        // 成功表示该生产者P1完成连续完成了T1、T2，队列处于S1
        // 失败表示T2已经由生产者P2完成，队列处于S1
        return true;
      }
      // 失败者得知队列处于S2，则尝试T2：CAS设置tail
      tail.compareAndSet(curTail, curNext.get());
      // 如果成功，队列转换到S1；如果失败，队列表示T2已经由生产者P1完成，队列已经处于S1
      // 然后循环，重新尝试T1
    }
  }

  private static class Node<E> {
    private volatile E item;
    private AtomicReference<Node<E>> next;

    public Node(E item, AtomicReference<Node<E>> next) {
      this.item = item;
      this.next = next;
    }
  }
}
```

#### 减少无效的竞争

我们涉及的状态比较少（只有2个状态），继续看看能否减少无效的竞争，比如：

* 前两种实现的第一步都是CAS尝试T1，失败了就退化成一次探查（compare and swap中的compare）。发起CAS前，可能队列已经处于S2，这时CAS尝试T1就成了浪费，只需要探查即可。这有点像DCL单例的思路([面试中单例模式有几种写法？](/2017/09/27/面试中单例模式有几种写法？/))，**可以直接通过tail.next判断队列是否处于S1，来完成一部分探查，以减少无效的竞争**。

```java
public class ConcurrentLinkedQueue3<E> {
  private AtomicReference<Node<E>> tail;

  public ConcurrentLinkedQueue3() {
    throw new UnsupportedOperationException("Not implement");
  }

  public boolean offer(E e) {
    Node<E> newNode = new Node<E>(e, new AtomicReference<>(null));
    while (true) {
      Node<E> curTail = tail.get();
      AtomicReference<Node<E>> curNext = curTail.next;

      // 先检查一下队列状态的状态，tail.next==null表示队列处于状态S1，仅此时才有CAS尝试T1的必要
      if (curNext.get() == null) {
        // 如果处于S1，尝试T1：CAS设置tail.next
        if (curNext.compareAndSet(null, newNode)) {
          // 成功者完成了T1，队列处于S2，继续尝试T2：CAS设置tail
          tail.compareAndSet(curTail, curNext.get());
          // 成功表示该生产者P1完成连续完成了T1、T2，队列处于S1
          // 失败表示T2已经由生产者P2完成，队列处于S1
          return true;
        }
      }
      // 否则队列处于处于S2，或CAS尝试T1的失败者得知队列处于S2，则尝试T2：CAS设置tail
      tail.compareAndSet(curTail, curNext.get());
      // 如果成功，队列转换到S1；如果失败，队列表示T2已经由生产者P1完成，队列已经处于S1
      // 然后循环，重新尝试T1
    }
  }

  private static class Node<E> {
    private volatile E item;
    private AtomicReference<Node<E>> next;

    public Node(E item, AtomicReference<Node<E>> next) {
      this.item = item;
      this.next = next;
    }
  }
}
```

>注意，上述实现中，while代码块后都没有返回值。这是被编译器允许的，因为编译器可以分析出，该方法不可能运行到while代码块之后，所以while代码块后的返回值语句也是无效的。

## 出队方法poll()

对偶的构造一个简化的场景：**存在2个消费者C1、C2，同时触发出队操作**。

不需要考虑悲观策略和优化方案，我们尝试基于思路2的第一种实现撸一版基础的poll()方法。

>然后，，，没撸动。想了一下，朴素链表（如LinkedList）中，直接用head表示维护头结点无法区分“已取出item未移动head指针”和“未取出item未移动head指针”（同“已取出item已移动head指针”）两种状态。所以还是**写一写才知道深浅**啊，碰巧前两天写了BlockingQueue的分析，dummy node正好派上用场。

队列初始化如下：

```java
dummy = new Node(null, null);
// tail = dummy; // 后面会用到
// head = dummy.next; // dummy.next 表示实际的头结点，但我们不需要存储它
```

### 状态机

单线程环境的出队过程：

```java
// 准备
curDummy = dummy;
curNext = curDummy.next;
oldItem = curNext.item;

// 出队前
assert dummy == curDummy && dummy.next.item == oldItem; // 状态S1
// 开始出队
dummy.next.item = null; // 事件E1
// 出队中
assert dummy == curDummy && dummy.next.item == null; // 状态S2
dummy = dummy.next; // 事件E2
// 结束出队
// 出队后
assert dummy == curNext && dummy.next.item != null; // 状态S3，合并到状态S1
```

状态：

* 状态S1：事件E1执行前，dummy指向实际的dummy节点curDummy，dummy.next.item== oldItem。如消费者C1、C2都还没有触发出队时，队列处于状态S1；消费者C1完成入队C2还没触发出队时，队列处于状态S1。
* 状态S2：事件E1执行后、E2执行前，dummy指向旧的dummy节点curDummy，dummy.next.item==null。
* ~~状态S3：事件E2执行后，dummy指向新的dummy节点curNext，dummy.next.item!=null。这在本质上同状态S1是一致的，合并。~~

状态转换：

* 状态转换T1：S1->S2，即dummy.next.item = null。
* 状态转换T2：S2->S1，即dummy = dummy.next。

### 代码

```java
public class ConcurrentLinkedQueue4<E> {
  private AtomicReference<Node<E>> dummy;

  public ConcurrentLinkedQueue4() {
    dummy = new AtomicReference<>(new Node<>(null, null));
  }

  public E poll() {
    while (true) {

      Node<E> curDummy = dummy.get();
      Node<E> curNext = curDummy.next;
      E oldItem = curNext.item.get();
      // 尝试T1：CAS设置dummy.next.item
      if (curNext.item.compareAndSet(oldItem, null)) {
        // 成功者完成了T1，队列处于S2，继续尝试T2：CAS设置dummy
        dummy.compareAndSet(curDummy, curNext);
        // 成功表示该消费者C1完成连续完成了T1、T2，队列处于S1
        // 失败表示T2已经由消费者C2完成，队列处于S1
        return oldItem;
      }
      // 失败者得知队列处于S2，则尝试T2：CAS设置dummy
      dummy.compareAndSet(curDummy, curNext);
      // 如果成功，队列转换到S1；如果失败，队列表示T2已经由消费者P1完成，队列已经处于S1
      // 然后循环，重新尝试T1
    }
  }

  private static class Node<E> {
    private AtomicReference<E> item;
    private volatile Node<E> next;

    public Node(AtomicReference<E> item, Node<E> next) {
      this.item = item;
      this.next = next;
    }
  }
}
```

### 另一种状态机

实际上，前面的讨论有意回避了一个问题——如果入队/出队操作顺序不同，我们会构造出不同的状态机。这相当于同一个类的另一种实现，不违反前面作出的声明：

>状态机模型与是否需要并发无关，一个类不管是否是线程安全的，其状态机模型从类被实现（此时，所有类行为都是确定的）开始就是确定的。

继续以出队为例，假设在单线程下，采用这样的顺序出队：

```java
// 准备
curDummy = dummy;
curNext = curDummy.next;
oldItem = curNext.item;

// 出队前
assert dummy == curDummy && dummy.item == null; // 状态S1

// 开始出队
dummmy = dummy.next; // 事件E1
// 出队中
assert dummy == curNext && dummy.item == oldItem; // 状态S2
dummy.item = null; // 事件E2
// 结束出队

// 出队后
assert dummy == curNext && dummy.item == null; // 状态S3，合并到状态S1
```

看起来，这样的操作顺序更容易定义各状态：

* 状态S1：事件E1执行前，dummy指向实际的dummy节点curDummy，dummy.item == null。如消费者C1、C2都还没有触发出队时，队列处于状态S1；消费者C1完成入队C2还没触发出队时，队列处于状态S1。
* 状态S2：事件E1执行后、E2执行前，dummy指向新的dummy节点curNext，dummy.item == oldItem。
* ~~状态S3：事件E2执行后，dummy指向新的dummy节点curNext，dummy.item == null。显然同状态S1，合并。~~

状态转换：

* 状态转换T1：S1->S2，即dummmy = dummy.next。
* 状态转换T2：S2->S1，即dummy.item = null。

实现如下：

```java
public class ConcurrentLinkedQueue5<E> {
  private AtomicReference<Node<E>> dummy;

  public ConcurrentLinkedQueue5() {
    dummy = new AtomicReference<>(new Node<>(null, null));
  }

  public E poll() {
    while (true) {

      Node<E> curDummy = dummy.get();
      Node<E> curNext = curDummy.next;
      E oldItem = curNext.item.get();
      // 尝试T1：CAS设置dummmy
      if (dummy.compareAndSet(curDummy, curNext)) {
        // 成功者完成了T1，队列处于S2，继续尝试T2：CAS设置dummy.item
        curDummy.item.compareAndSet(oldItem, null);
        // 成功表示该消费者C1完成连续完成了T1、T2，队列处于S1
        // 失败表示T2已经由消费者C2完成，队列处于S1
        return oldItem;
      }
      // 失败者得知队列处于S2，则尝试T2：CAS设置dummy.item
      curDummy.item.compareAndSet(oldItem, null);
      // 如果成功，队列转换到S1；如果失败，队列表示T2已经由消费者P1完成，队列已经处于S1
      // 然后循环，重新尝试T1
    }
  }

  private static class Node<E> {
    private AtomicReference<E> item;
    private volatile Node<E> next;

    public Node(AtomicReference<E> item, Node<E> next) {
      this.item = item;
      this.next = next;
    }
  }
}
```

### 一个trick

实现上面状态机的过程中，我想出了一个针对出队操作的trick：**可以去掉dummy node，用head维护头结点+一步状态转换完成出队**。

>对啊，我写着写着又撸出来了。。。

去掉了dummy node，那么head.item的初始状态就是非空的，下面是简化的状态机。

单线程出队的操作顺序：

```java
// 准备
curHead = head;
curNext = curHead.next;
oldItem = curHead.item;

// 出队前
assert head == curHead; // 状态S1

// 出队
head = head.next; // 事件E1

// 出队后
assert head == curNext; // 状态S2，合并到状态S1
```

出队只需要尝试head后移，成功者可从旧的头结点curHead中取出item，之后curHead将被废弃；失败者再重新尝试即可。如果在尝试前就得到了item的引用，那么E1发生后，不管成功与否，在curHead上做什么都是无所谓的了，因为事实上没有任何消费者会再去访问它。

这是一个单状态的状态机，则状态：

* 状态S1：head指向实际的头节点curHead。队列始终处于状态S1。
* ~~状态S2：head指向新的头节点curNext。同S1，合并~~

状态转换：

* 状态转换T1：S1->S1，即head = head.next。

实现如下：

```java
public class ConcurrentLinkedQueue6<E> {
  private AtomicReference<Node<E>> head;

  public ConcurrentLinkedQueue6() {
    throw new UnsupportedOperationException("Not implement");
  }

  public E poll() {
    while (true) {

      Node<E> curHead = head.get();
      Node<E> curNext = curHead.next;
      // 尝试T1：CAS设置head
      if (head.compareAndSet(curHead, curNext)) {
        // 成功者完成了T1，队列处于S1
        return curHead.item; // 只让成功者取出item
      }
      // 失败者重试尝试
    }
  }

  private static class Node<E> {
    private volatile E item;
    private volatile Node<E> next;

    public Node(E item, Node<E> next) {
      this.item = item;
      this.next = next;
    }
  }
}
```

## 其他特殊情况

前面都是基于假设2“入队、出队无竞争”讨论的。现在需要放开假设2，看如何完善已有的实现以保证假设2成立。或者如果不能保证假设2的话，如何解决竞争问题。

根据对LinkedBlockingQueue的分析，我们得知，如果底层数据结构是朴素链表，那么队列空或长度为1的时候，head、tail都指向同一个节点（或都为null）,这时必然存在竞争；dummy node较好的解决了这一问题。ConcurrentLinkedQueue4是基于dummy node的方案，我们尝试在此基础上修改。

回顾dummy node的使用方法（配合ConcurrentLinkedQueue2和ConcurrentLinkedQueue4做了调整和精简）:

* 初始化链表时，创建dummy node：
    * dummy = new Node(null, null)
    * // head = dummy.next // head 为 null <=> 队列空
    * tail = dummy // tail.item 为 null <=> 队列空
* 在队尾入队时，tail后移：
    * tail.next = new Node(newItem, null)
    * tail = tail.next
* 在队头出队时，dummy后移，同步更新head：
    * oldItem = dummy.next.item // == head.item
    * dummy.next.item = null
    * dummy = dummy.next
    * // head = dummy.next
    * return oldItem

下面分情况讨论。

### case1：队列空

队列空时，队列处于一个特殊的状态，从该状态出发，仅能完成入队相关的状态转换——通俗讲就是队列空时只允许入队操作。这时消除竞争很简单，只允许入队不允许出队即可：

```java
public class ConcurrentLinkedQueue7<E> {
  private AtomicReference<Node<E>> dummy;
  private AtomicReference<Node<E>> tail;

  public ConcurrentLinkedQueue7() {
    Node<E> initNode = new Node<E>(
        new AtomicReference<E>(null), new AtomicReference<Node<E>>(null));
    dummy = new AtomicReference<>(initNode);
    tail = new AtomicReference<>(initNode);
    // Node<E> head = dummy.get().next.get();
  }

  public boolean offer(E e) {
    Node<E> newNode = new Node<E>(new AtomicReference<>(e), new AtomicReference<>(null));
    while (true) {
      Node<E> curTail = tail.get();
      AtomicReference<Node<E>> curNext = curTail.next;
      if (curNext.compareAndSet(null, newNode)) {
        tail.compareAndSet(curTail, curNext.get());
        return true;
      }
      tail.compareAndSet(curTail, curNext.get());
    }
  }

  public E poll() {
    while (true) {
      Node<E> curDummy = dummy.get();
      Node<E> curNext = curDummy.next.get();
      // 既可以用 dummy.next == null (head) 判空，也可以用 tail.item == null
      // 不过鉴于处于poll()方法中，使用 dummy.next 可读性更好
      if (curNext == null) {
        return null;
      }
      E oldItem = curNext.item.get();
      if (curNext.item.compareAndSet(oldItem, null)) {
        dummy.compareAndSet(curDummy, curNext);
        return oldItem;
      }
      dummy.compareAndSet(curDummy, curNext);
    }
  }

  private static class Node<E> {
    private AtomicReference<E> item;
    private AtomicReference<Node<E>> next;

    public Node(AtomicReference<E> item, AtomicReference<Node<E>> next) {
      this.item = item;
      this.next = next;
    }
  }
}
```

ConcurrentLinkedQueue7需要原子的操作item和next，因此Node的item、next域都被声明为了AtomicReference。

队列空的时候：offer()方法同ConcurrentLinkedQueue2#offer()，不需要做特殊处理；poll()方法在ConcurrentLinkedQueue4#poll()的基础上，增加了32-34行的队列空检查。需要注意的是，检查必须放在队列转换的过程中，防止消费者C2第一次尝试时队列非空，但第二次尝试时队列变空（由于C1取出了唯一的元素）的情况。

### case2：队列长度等于1

队列长度等于1时，入队与出队不会同时修改同一节点，这时一定不会发生竞争。分析如下。

假设存在一个生产者P1，一个消费者C1，同时触发入队/出队，队列中只有一个元素，所以只两个节点dummyNode、singleNode则此时：

```
assert dummy == dummyNode;
assert dummy.next.item == singleNode.item;

assert tail == singleNode;
assert tail.next == singleNode.next;
```

回顾ConcurrentLinkedQueue7的实现:

* poll()方法修改引用dummy、singleNode.item
* offer()方法操tail、singleNode.next

因此，由于dummy node的引入，队列长度为1时，入队、出队之间天生就不存在竞争。

## 小结

至此，我们从最简单的场景触发，基于状态机实现了一个支持高性能offer()、poll()方法的ConcurrentLinkedQueue7。CAS的好处暂且不表，重要的是基于状态机进行并发程序设计的思想。只有抓住其状态机的本质，才能设计出正确、高效的并发类。

>如果还是没有体会到状态机的精妙之处，可以抛开状态机，并自己尝试基于乐观策略实现ConcurrentLinkedQueue。（之所以要基于乐观策略，是因为悲观策略可以认为是乐观策略的是特例，容易让人忽略其状态机的本质）

# JDK实现

>希望看到这里，你已经理解了ConcurrentLinkedQueue的状态机本质，因为下面就不再是本文的重点。

真·神Doug Lea的实现基于一个弱一致性的状态机：允许队列处于多种不一致的状态，通过恰当的选择“不一致的状态”，能做到用户无感；虽然增加了状态机的复杂度，但也进一步提高了性能。

网上分析文章非常多，读者可自行阅读，有一定难度。**本文不打算讲解Doug Lea的实现，贴出源码仅供大家膜拜**。

## 构造方法

常用的是默认的空构造函数：

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>, java.io.Serializable {
...
    private transient volatile Node<E> head;
    private transient volatile Node<E> tail;
    public ConcurrentLinkedQueue() {
        head = tail = new Node<E>(null);
    }
...
}
```

Doug Lea也使用了dummy node，不过命名为了head。初始化方法同我们实现的ConcurrentLinkedQueue7。

## 入队方法offer()

ConcurrentLinkedQueue7#offer()相当于ConcurrentLinkedQueue#offer()的一个特例。

```java
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            if (p.casNext(null, newNode)) {
                if (p != t)
                    casTail(t, newNode);
                return true;
            }
        }
        else if (p == q)
            p = (t != (t = tail)) ? t : head;
        else
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

具体来讲，ConcurrentLinkedQueue允许的多个状态大体是这样的：

* 状态S1：一致；newNode已衔接在tail.next，但tail指向倒数第1个节点
* 状态S2：不一致；newNode已衔接在tail.next，但tail指向倒数第2个节点
* 状态S3：不一致；newNode已衔接在tail.next，但tail指向倒数第3个节点
* ...

>状态转换的规则也随之打破——不再需要连续完成T1、T2，可以**连续执行多次类T1，最后执行一次类T2**。

for循环中的几个分支就是在处理这些一致和不一致的状态。我们前面定义的状态机空间中只允许状态S1、S2，因此是一个子集。增加的这些不一致的状态主要是为了减少CAS次数，进一步提高队列性能，这包含两个重要意义：

* 降低延迟：部分入队请求不再需要走完完整的状态转换，只需要循环到tail.next.cas(null, newNode)成功。
* 提高吞吐：之前每一次入队请求都要设置一次tail节点；目前只需要积攒几次入队，在某个新的newNode入队时，直接尝试tail.cas(t, newNode)，将tail跳跃到最新的newNode。

>增加这些不一致的状态是很危险的，如S3，当队列长度为1的时候，tail与head的位置存在交叉。Doug Lea牛逼之处在于，在保证正确性的前提下，不仅通过增加状态提高了性能，还减少了实际的CAS次数。

## 出队方法poll()

```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;

            if (item != null && p.casItem(item, null)) {
                if (p != h)
                    updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
            else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}
final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
        h.lazySetNext(h);
}
```

分析方法类似于offer()。注意下updateHead()。

### 未完

本来是想分析ConcurrentLinkedQueue源码的，没想到写完状态机就3600多字了，干货却不多。前路漫漫，源码咱下回见。
