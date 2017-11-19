---
title: 源码|并发一枝花之CopyOnWriteArrayList
tags:
  - Java
  - 并发
  - 面试
  - 原创
reward: true
date: 2017-10-24 22:33:46
---

CopyOnWriteArrayList的设计思想非常简单，但在设计层面有一些小问题需要注意。

<!--more-->

>JDK版本：oracle java 1.8.0_102

>本来不想写的，但是github上CopyOnWriteArrayList的code results也有165k，为了流量还是写一写吧。

# 实现

看两个方法你就懂了。

## 读元素set()

```java
public E get(int index) {
    return get(getArray(), index);
}
final Object[] getArray() {
    return array;
}
```

get()方法直接调用内部的getArray()方法，而getArray()方法则直接返回成员变量array。

>我没明白为什么要再封装一层，而不是直接访问。

array指向一个数组，是CopyOnWriteArrayList的内部数据结构：

```java
private transient volatile Object[] array;
```

敲黑板！！！
 
**array是一个volatile变量，**其读、写操作具有Happends-Before关系。具体来讲，线程W1通过set()方法“修改”集合后，线程R1能立刻通过get()方法得到array的最新值。

>你可以理解为volatile变量的读、写是原子的，不过，我更希望你能从顺序和可见性的角度理解理解volatile、锁等具有偏序关系的操作。volatile的原理和用法见[volatile关键字的作用、原理](/volatile%E5%85%B3%E9%94%AE%E5%AD%97%E7%9A%84%E4%BD%9C%E7%94%A8%E3%80%81%E5%8E%9F%E7%90%86/)。

## 写元素set()

重点是set()方法：

```java
public E set(int index, E element) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        E oldValue = get(elements, index);

        if (oldValue != element) {
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len);
            newElements[index] = element;
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            setArray(elements);
        }
        return oldValue;
    } finally {
        lock.unlock();
    }
}
final void setArray(Object[] a) {
    array = a;
}
```

set()方法也很简单，两个要点：

1. 通过锁lock保护队列修改过程
2. 在副本上修改，最后替换array引用

按照独占锁的思路，仅仅给写线程加锁是不行的，会有读、写线程的竞争问题。但是get()中明明没有加锁，为什么也没有问题呢？

通过加锁，保证同一时间最多只有一个写线程W1进入try block；假设要设置的值与旧值不同。9-10行首先将数据复制一份（此时，没有其他写线程能进入try block修改集合），11行**在副本上修改相应元素**，12行修改array引用。array是volatile变量，所以写的最新值对其他读线程、写线程都是可见的。

这就是所谓的“`写时复制`”。

### 其他问题

#### 15行volatile写的作用

实际上，**15行的volatile写是多余的**。这只是为了能从代码里理解到volatile写的语义，并不必要的保证什么——不过这种考虑也是不恰当的，反而使代码迷惑。一个类似的例子是addIfAbsent()：

```java
public boolean addIfAbsent(E e) {
    Object[] snapshot = getArray();
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
        addIfAbsent(e, snapshot);
}
private boolean addIfAbsent(E e, Object[] snapshot) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) {
            // Optimize for lost race to another addXXX operation
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i]))
                    return false;
            if (indexOf(e, current, common, len) >= 0)
                    return false;
        }
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

基本思想相同，17、19行都是直接返回，并没有做多余的“volatile写”。

在网上搜的话，还有很多其他观点。如果你认为我的观点是错误的，欢迎交流。

> addIfAbsent的编码风格跟set()区别很大，不像一个人写的。需要认识到，JDK是一个发展、变化的产品，一个包、甚至一个类都可能不是同一个人、同一段时间写的，编码风格、设计思想可能发生变化；更不要假定JDK的实现一定是对的（当然，绝大部分时候是对的），要基于正确的逻辑去分析，再做判断。

#### 为什么必须要给set加锁？

>TODO 20171024

看起来，如果不给set加锁，似乎并发性能更高，一致性也没有削弱多少。未解决，欢迎交流。

# 设计思想

最后总结CopyOnWriteArrayList的设计思想：

* 用并发访问“数组副本的引用”代替并发访问“数组元素的引用”，大大降低了维护线程安全的难度。
* 当前副本可能是失效的，但一定是集合在某一瞬间的快照（一定程度上满足不变性），满足弱一致性。
