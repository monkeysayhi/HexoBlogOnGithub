---
title: CAS中的ABA问题
tags:
  - Java
  - 并发
  - 面试
  - 原创
reward: true
date: 2018-01-02 14:53:11
---

补档CAS中的ABA问题。

要特别注意，常见的ABA问题有两种，要求能分别举例解释。

<!--more-->

# 1 定义

## 1.1 基本的ABA问题

在CAS算法中，需要取出内存中某时刻的数据（由用户完成），在下一时刻比较并替换（由CPU完成，该操作是原子的）。这个时间差中，会导致数据的变化。

假设如下事件序列：

1. 线程 1 从内存位置V中取出A。
1. 线程 2 从位置V中取出A。
1. 线程 2 进行了一些操作，将B写入位置V。
1. 线程 2 将A再次写入位置V。
1. 线程 1 进行CAS操作，发现位置V中仍然是A，操作成功。

尽管线程 1 的CAS操作成功，但不代表这个过程没有问题——_对于线程 1 ，线程 2 的修改已经丢失_。

# 1.2 与内存模型相关的ABA问题

在没有垃圾回收机制的内存模型中（如C++），程序员可随意释放内存。

假设如下事件序列：

1. 线程 1 从内存位置V中取出A，A指向内存位置W。
1. 线程 2 从位置V中取出A。
1. 线程 2 进行了一些操作，释放了A指向的内存。
1. 线程 2 重新申请内存，并恰好申请了内存位置W，将位置W存入C的内容。
1. 线程 2 将内存位置W写入位置V。
1. 线程 1 进行CAS操作，发现位置V中仍然是A指向的即内存位置W，操作成功 

这里比问题 1.1 的后果更严重，实际内容已经被修改了，但_线程 1 无法感知到线程 2 的修改_。

更甚，如果线程 2 只释放了A指向的内存，而线程 1 在 CAS之前还要访问A中的内容，那么线程 1 将访问到一个`野指针`。

# 2 举例

## 2.1 基本的ABA问题举例

如果位置V存储的是链表的头结点，那么发生ABA问题的链表中，原头结点是node1，线程 2 操作头结点变化了两次，很可能是先修改头结点为node2，再将node1（在C++中，也可是重新分配的节点node3，但恰好其指针等于已经释放掉的node1）插入表头成为新的头结点。

对于线程 1 ，头结点仍旧为 node1（或者说头结点的值，因为在C++中，虽然地址相同，但其内容可能变为了node3），CAS操作成功，但头结点之后的子链表的状态已不可预知。

>脑补示意图。。

## 2.2 与内存模型相关的ABA问题举例

问题定义已阐述清楚。

# 3 解决

Java的垃圾回收机制已经帮我们解决了问题 1.2；至于问题 1.1，加入版本号即可解决。

## 3.1 AtomicStampedReference

除了对象值，AtomicStampedReference内部还维护了一个“`状态戳`”。状态戳可类比为时间戳，是一个整数值，_每一次修改对象值的同时，也要修改状态戳，从而区分相同对象值的不同状态_。当AtomicStampedReference设置对象值时，对象值以及状态戳都必须满足期望值，写入才会成功。

AtomicStampedReference的几个API在AtomicReference的基础上新增了有关时间戳的信息：

```java
//比较设置 参数依次为：期望值 写入新值 期望时间戳 新时间戳
public boolean compareAndSet(V expectedReference, V newReference, 
    int expectedStamp, int newStamp)
//获得当前对象引用
public V getReference()
//获得当前时间戳
public int getStamp()
//设置当前对象引用和时间戳
public void set(V newReference, int newStamp)
```

## 3.2 AtomicMarkableReference

AtomicMarkableReference和AtomicStampedReference功能相似，但**AtomicMarkableReference描述更加简单的是与否的关系**。它的定义就是将状态戳简化为`true|false`。如下：

```java
public final static AtomicMarkableReference<String> ATOMIC_MARKABLE_REFERENCE 
    = new AtomicMarkableReference<>("abc" , false); 
```

操作时：

```java
ATOMIC_MARKABLE_REFERENCE.compareAndSet("abc", "abc2", false, true);
```
