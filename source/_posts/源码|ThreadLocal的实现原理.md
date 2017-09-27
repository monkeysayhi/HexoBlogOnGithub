---
title: 源码|ThreadLocal的实现原理  
date: 2016-11-27 23:45:05  
tags: 
 - Java
 - 并发
 - 面试
 - 原创
reward: true  
---

ThreadLocal也叫“线程本地变量”、“线程局部变量”：

* 其作用域覆盖线程，而不是某个具体任务；
* 其“自然”的生命周期与线程的生命周期“相同”（但在JDK实现中比线程的生命周期更短，减少了内存泄漏的可能）。

ThreadLocal代表了一种**线程与任务剥离**的思想，从而达到`线程封闭`的目的，帮助我们设计出更“健康”（简单，美观，易维护）的线程安全类。  

<!--more-->

# 一种假象
ThreadLocal的使用方法往往给我们造成一种假象——变量封装在任务对象内部。
## 根据使用方法推测实现思路
我们在使用ThreadLocal对象的时候，往往是在任务对象内部声明ThreadLocal变量，如：  

```java
ThreedLocal<List> onlineUserList = …;
```

这也是此处说“从概念上”可以视作如此的原因。那么从这种使用方法的角度出发（逆向思维），让我们自然而然的认为，**ThreadLocal变量是存储在任务对象内部的**，那么实现思路如下：  

```java
class ThreadLocal<T>{
	private Map<Thread, T> valueMap = …;
	public T get(Object key){
		T realValue = valueMap.get(Thread.currentThread())
		return realValue;
	}
}
```

## 存在问题
但是这样实现存在一个问题：

* 线程死亡之后，任务对象可能仍然存在（这才是最普遍的情况），从而ThreadLocal对象仍然存在。我们不能要求线程在死亡之前主动删除其使用的ThreadLocal对象，所以valueMap中该线程对应的entry（<key, realValue>）无法回收；

问题的本质在于，这种实现*“将线程相关的域封闭于任务对象，而不是线程中”*。所以ThreadLocal的实现中最重要的一点就是——**“将线程相关的域封闭在当前线程实例中”**，虽然域仍然在任务对象中声明、set和get，却与任务对象无关。  
# 源码分析
下面从源码中分析ThreadLocal的实现。  
## 逆向追踪源码
首先观察看ThreadLocal类的源码，找到构造函数：  

```java
    /**
     * Creates a thread local variable.
     * @see #withInitial(java.util.function.Supplier)
     */
    public ThreadLocal() {
    }
```

空的。  
直接看set方法：  

```java
    /**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value); // 使用this引用作为key，既做到了变量相关，又满足key不可变的要求。
        else
            createMap(t, value);
    }
```

设置value时，要根据当前线程t获取一个ThreadLocalMap类型的map，真正的value保存在这个map中。这验证了之前的一部分想法——ThreadLocal变量保存在一个`“线程相关”的map`中。
进入getMap方法：  

```java
    /**
     * Get the map associated with a ThreadLocal. Overridden in
     * InheritableThreadLocal.
     *
     * @param  t the current thread
     * @return the map
     */
    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }
```

可以看到，该map实际上保存在一个Thread实例中，也就是之前传入的当前线程t。  
观察Thread类的源码，确实存在着threadLocals变量的声明：  

```java
    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
```
在这种实现中，ThreadLocal变量已经达到了文章开头的提出的基本要求：

>* 其作用域覆盖线程，而不是某个具体任务
* 其“自然”的生命周期与线程的生命周期“相同”

如果是面试的话，一般分析到这里就可以结束了。  

## 进阶
希望进一步深入，可以继续查看ThreadLocal.ThreadLocalMap类的源码：  

```java
    /**
     * ThreadLocalMap is a customized hash map suitable only for
     * maintaining thread local values. No operations are exported
     * outside of the ThreadLocal class. The class is package private to
     * allow declaration of fields in class Thread.  To help deal with
     * very large and long-lived usages, the hash table entries use
     * WeakReferences for keys. However, since reference queues are not
     * used, stale entries are guaranteed to be removed only when
     * the table starts running out of space.
     */
    static class ThreadLocalMap {

        /**
         * The entries in this hash map extend WeakReference, using
         * its main ref field as the key (which is always a
         * ThreadLocal object).  Note that null keys (i.e. entry.get()
         * == null) mean that the key is no longer referenced, so the
         * entry can be expunged from table.  Such entries are referred to
         * as "stale entries" in the code that follows.
         */
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k); // 使用了WeakReference中的key
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;
```

可以看到，虽然ThreadLocalMap没有实现Map接口，但也具有常见Map实现类的大部分属性（与HashMap不同，hash重复时在table里顺序遍历）。  
重要的是Entry的实现。Entry继承了一个ThreadLocal泛型的WeakReference引用。  

1. ThreadLocal说明了Entry的类型，保存的是ThreadLocal变量。
2. WeakReference是为了方便垃圾回收。

### WeakReference与内存泄露
#### 仍然存在的内存泄露
现在，我们已经很好的*“将线程相关的域封闭在当前线程实例中”*，如果线程死亡，线程中的ThreadLocalMap实例也将被回收。  
看起来一切都那么美好，但我们忽略了一个很严重的问题——**如果任务对象结束而线程实例仍然存在（常见于线程池的使用中，需要复用线程实例），那么仍然会发生内存泄露**。我们可以建议广大Javaer手动remove声明过的ThreadLocal变量，但这种回归C++的语法是不能被Javaer接受的；另外，**要求我猿记住声明过的变量，简直比约妹子吃饭还困难**。  
#### 使用WeakReference减少内存泄露
对ThreadLocal源码的分析让我第一次了解到WeakReference的作用的使用方法。

>对于弱引用WeakReference，当一个对象仅仅被弱引用指向, 而没有任何其他强引用StrongReference指向的时候, 如果GC运行, 那么这个对象就会被回收。  

在ThreadLocal变量的使用过程中，由于只有任务对象拥有ThreadLocal变量的强引用（考虑最简单的情况），所以*任务对象被回收后，就没有强引用指向ThreadLocal变量*，ThreadLocal变量也就会被回收。  
之所以这里说“减少内存泄露”，是因为单纯使用WeakReference仅仅解决了问题的前半部分。
#### 进一步减少内存泄露
尽管现在使用了弱引用，ThreadLocalMap中仍然会发生内存泄漏。原因很简单，ThreadLocal变量只是Entry中的key，所以**Entry中的key虽然被回收了，Entry本身却仍然被引用**。  
为了解决这后半部分问题，*ThreadLocalMap在它的getEntry、set、remove、rehash等方法中都会主动清除ThreadLocalMap中key为null的Entry*。  
这样做已经可以大大减少内存泄露的可能，但**如果我们声明ThreadLocal变量后，再也没有调用过上述方法，依然会发生内存泄露**。不过，现实世界中线程池的容量总是有限的，所以这部分泄露的内存并不会无限增长；另一方面，一个大量线程长期空闲的线程池（这时内存泄露情况可能比较严重），也自然没有存在的必要，而一旦使用了线程，泄露的内存就能够被回收。因此，*我们通常认为这种内存泄露是可以“忍受”的*。  
同时应该注意到，这里将ThreadLocal对象“自然”的生命周期**“收紧”**了一些，从而比线程的生命周期更短。  
#### 其他内存泄露情况
还有一种较通用的内存泄露情况：**使用static关键字声明变量**。  
使用static关键字延长了变量的生命周期，可能导致内存泄露。对于ThreadLocal变量也是如此。  
更多内存泄露的分析可参见[ThreadLocal 内存泄露的实例分析](http://blog.xiaohansong.com/2016/08/09/ThreadLocal-leak-analyze/)，这里不再展开。  

>参考：  
[ThreadLocal 源码剖析](http://www.cnblogs.com/digdeep/p/4510875.html)  
[深入分析 ThreadLocal 内存泄漏问题](http://blog.xiaohansong.com/2016/08/06/ThreadLocal-memory-leak/)  
