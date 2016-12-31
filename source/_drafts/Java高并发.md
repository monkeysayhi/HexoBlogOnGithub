---
title: Java高并发  
tags: [Java,并发,面试,原创]  
reward: true  
---

曾经，我在面试Java研发实习生时最常听到的一句话就是：  

>搞Java怎么能不学并发呢？

没错，真的是经过了面试官的无数鄙视，我才知道Java并发编程在Java语言中的重要性。

<!--more-->

# 并发模型
## 悲观锁和乐观锁的理解及如何实现，有哪些实现方式？
## 悲观锁
悲观锁假设最坏的情况（如果你不锁门，那么捣蛋鬼就会闯入并搞得一团糟），并且只有在确保其他线程不会干扰（通过获取正确的锁）的情况下才能执行下去。  
常见实现如独占锁等。  
安全性更高，但在中低并发程度下的效率更低。  
## 乐观锁
乐观锁借助冲突检查机制来判断在更新过程中是否存在其他线程的干扰，如果存在，这个操作将失败，并且可以重试（也可以不重试）。  
常见实现如CAS等。  
部分乐观锁削弱了一致性，但中低并发程度下的效率大大提高。
# 并发编程
## Java中如何创建一个线程
从面相接口的角度上讲，实际上只有一种方法实现Runable接口；但Thread类为线程操作提供了更多的支持，所以通常做法是实现Runable接口，实例化并传入Throughead类的构造函数。

1. 继承Thread，覆写run方法
2. 实现Runable接口，覆写run方法
3. 为了管理多线程，分离线程管理和业务逻辑，引入了线程池ThreadPool：  
*无限制的创建线程会引起应用程序内存溢出。所以创建一个线程池是个更好的的解决方案，因为可以限制线程的数量并且可以回收再利用这些线程。*  

## Vector（HashTable）如何实现线程安全
通过synchronized关键字修饰每个方法。  
依据synchronized关键字引申出以下问题。  
## synchronized修饰方法和修饰代码块时有何不同
持有锁的对象不同：  

1. 修饰方法时：this引用的当前实例持有锁  
2. 修饰代码块时：要指定一个对象，该对象持有锁  

从而导致二者的意义不同：

1. 同步代码块在锁定的范围上可能比同步方法要小，一般来说锁的范围大小和性能是成反比的。
2. 修饰代码块可以选择对哪个对象加锁，但是修饰方法只能给this对象加锁。

## ConcurrentHashMap的如何实现线程安全
ConcurrentHashMap的线程安全实现与HashTable不同：

1. 可以将ConcurrentHashMap理解为，不直接持有一个HashMao，而是用多个Segment代替了一个HashMap。但实际实现的Map部分和HashMap的原理基本相同，对脚标取模来确定table[i]所属段，从而对不同的段获取不同的段锁。
2. 每个Segment持有一个锁，通过分段加锁的方式，既实现了线程安全，又兼顾了性能

## Java中有哪些实现并发编程的方法
要从最简单的答起，业界最常用的是重点，有新意就放在最后。   

1. synchronized关键字
2. 使用继承自Object类的wait、notify、notifyAll方法
3. 使用线程安全的API和集合类：  
	1. 使用Vector、HashTable等线程安全的集合类
	2. 使用Concurrent包中提供的ConcurrentHashMap、CopyOnWriteArrayList、ConcurrentLinkedQueue等弱一致性的集合类
	3. 在Collections类中有多个静态方法，它们可以获取通过同步方法封装非同步集合而得到的集合,如`List list = Collection.synchronizedList(new ArrayList())`。
	4. 使用原子变量、volatile变量等
4. 使用Concurrent包中提供的信号量Semaphore、闭锁Latch、栅栏Barrier、交换器Exchanger、Callable&Future、阻塞队列BlockingQueue等. 
5. 手动使用Lock实现基于锁的并发控制
6. 手动使用Condition或AQS实现基于条件队列的并发控制
7. 使用CAS和SPIN等实现非阻塞的并发控制
8. 使用不变类
9. 其他并发模型还没有涉及

从而引申出如下问题：
## ConcurrentHashMap的的实现原理（见前）
## CopyOnWriteArrayList的复制操作发生在什么时机
## synchronizedList&Vector的区别

1. synchronizedList的实现中，synchronized关键字修饰代码块；Vector的实现中修饰方法。
2. synchronizedList只封装了add、get、remove等代码块，但Iterator却不是同步的，**进行遍历时要手动进行同步处理**；Vector中对Iterator也进行了加锁。
3. synchronizedList能够将所有List实现类封装为同步集合，其内部持有的仍然是List的实现类（ArrayList/LinkedList），所以除同步外，几乎只有该实现类和Vector的区别。

## synchronized修饰方法和修饰代码块时有何不同（见前）
## 信号量Semaphore、闭锁Latch、栅栏Barrier、交换器Exchanger、Callable&Future、阻塞队列BlockingQueue的实现原理
## ConcurrentLinkedQueue的插入算法
算法核心可概括为两步：

1. 先检测是否是中间状态（SPIN）
2. 再尝试CAS插入

详细待补充。

>参考：  
[CopyOnWriteArrayList与Collections.synchronizedList的性能对比](http://blog.csdn.net/yangzl2008/article/details/39456817)  
[SynchronizedList和Vector的区别
](http://www.hollischuang.com/archives/498)  

## 在java中wait和sleep方法的不同？
最大的不同是在等待时wait会释放锁，而sleep一直持有锁。Wait通常被用于线程间交互，sleep通常被用于暂停执行。  
## 为什么wait, notify 和 notifyAll这些方法不在thread类里面？
主要原因是JAVA提供的锁是对象级的而不是线程级的，每个对象都有锁，通过线程获得。由于wait，notify和notifyAll都是锁级别的操作，所以把他们定义在Object类中因为锁属于对象。  
## 为什么wait和notify方法要在同步块中调用？
Java API强制要求这样做，如果你不这么做，你的代码会抛出IllegalMonitorStateException异常。还有一个原因是为了避免wait和notify之间产生竞态条件。
## 为什么你应该在循环中检查等待条件?
处于等待状态的线程可能会收到错误警报和伪唤醒，如果不在循环中检查等待条件，程序就会在没有满足结束条件的情况下退出。  
## Java线程池中submit() 和 execute()方法有什么区别？
两个方法都可以向线程池提交任务，execute()方法的返回类型是void，它定义在Executor接口中, 而submit()方法可以返回持有计算结果的Future对象，它定义在ExecutorService接口中，它扩展了Executor接口，其它线程池类像ThreadPoolExecutor和ScheduledThreadPoolExecutor都有这些方法。  
## volatile 变量和 atomic 变量有什么不同？
Volatile变量可以确保先行关系，即写操作会发生在后续的读操作之前, 但它并不能保证原子性。例如用volatile修饰count变量那么 count++ 操作就不是原子性的。而AtomicInteger类提供的atomic方法可以让这种操作具有原子性如getAndIncrement()方法会原子性的进行增量操作把当前值加一，其它数据类型和引用变量也可以进行相似操作。  
## 为什么Thread类的sleep()和yield ()方法是静态的？
Thread类的sleep()和yield()方法将在当前正在执行的线程上运行。所以在其他处于等待状态的线程上调用这些方法是没有意义的。这就是为什么这些方法是静态的。它们可以在当前正在执行的线程中工作，并避免程序员错误的认为可以在其他非运行线程调用这些方法。  
