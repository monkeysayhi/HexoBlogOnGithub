---
title: 源码|使用FutureTask的正确姿势
tags:
  - Java
  - 并发
  - 面试
  - 猿格
  - 原创
reward: true
date: 2017-10-29 20:40:26
---

线程池的实现核心之一是FutureTask。在提交任务时，用户实现的Callable实例task会被包装为FutureTask实例ftask；提交后任务异步执行，无需用户关心；当用户需要时，再调用FutureTask#get()获取结果——或异常。

随之而来的问题是，**如何优雅的获取ftask的结果并处理异常？**本文讨论使用FutureTask的正确姿势。

<!--more-->

>JDK版本：oracle java 1.8.0_102

今天换个风格。

# 源码分析

从提交一个Callable实例task开始。

## submit()

ThreadPoolExecutor直接继承AbstractExecutorService的实现。

```java
public abstract class AbstractExecutorService implements ExecutorService {
...
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task);
        execute(ftask);
        return ftask;
    }
...
}
```

后续流程可参考[源码|从串行线程封闭到对象池、线程池](/2017/10/26/源码%7C从串行线程封闭到对象池、线程池/)。最终会在ThreadPoolExecutor#runWorker()中执行task.run()。

task即5行创建的ftask，看newTaskFor()。

## newTaskFor()

AbstractExecutorService#newTaskFor()创建一个RunnableFuture类型的FutureTask。

```java
public abstract class AbstractExecutorService implements ExecutorService {
...
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }
...
}
```

看FutureTask的实现。

## FutureTask

```java
public class FutureTask<V> implements RunnableFuture<V> {
...
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
...
    public FutureTask(Callable<V> callable) {
        if (callable == null)
            throw new NullPointerException();
        this.callable = callable;
        this.state = NEW;       // ensure visibility of callable
    }
...
}
```

构造方法的重点是初始化ftask状态为NEW。

### 状态机

状态转换比较少，直接给状态序列：

```
* NEW -> COMPLETING -> NORMAL
* NEW -> COMPLETING -> EXCEPTIONAL
* NEW -> CANCELLED
* NEW -> INTERRUPTING -> INTERRUPTED
```

状态在后面有用.

### run()

简化如下：

```java
public class FutureTask<V> implements RunnableFuture<V> {
...
    public void run() {
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                V result;
                boolean ran;
                try {
                    result = c.call();
                    ran = true;
                } catch (Throwable ex) {
                    result = null;
                    ran = false;
                    setException(ex);
                }
                if (ran)
                    set(result);
            }
        } finally {
            runner = null;
            int s = state;
            if (s >= INTERRUPTING)
                handlePossibleCancellationInterrupt(s);
        }
    }
...
}
```

#### 如果执行时未抛出异常

如果未抛出异常，则ran==true，FutureTask#set()设置结果。

```java
public class FutureTask<V> implements RunnableFuture<V> {
...
    protected void set(V v) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = v;
            UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state
            finishCompletion();
        }
    }
...
}
```

* **outcome中保存结果result**
* 连续两步设置状态到NORMAL
* finishCompletion()执行一些清理

记住outcome。

>相当于4行获取独占锁，5-6行执行锁中的操作（注意，7行是不加锁的）。

#### 如果执行时抛出了异常

如果运行时抛出了异常，则被12行catch捕获，FutureTask#setException()设置结果；同时，ran==false，因此不执行FutureTask#set()。

```java
public class FutureTask<V> implements RunnableFuture<V> {
...
    protected void setException(Throwable t) {
        if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
            outcome = t;
            UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state
            finishCompletion();
        }
    }
...
}
```

* **outcome中保存异常t**
* 连续两步设置状态到EXCEPTIONAL
* finishCompletion()执行一些清理

如果没有抛出异常在，则outcome记录正常结果；如果抛出了异常，则outcome记录异常。

>如果认为正常结果和异常都属于“任务的输出”，则使用相同的变量outcome记录是合理的；同时，使用不同的结束状态区分outcome中记录的内容。

#### run()小结

FutureTask将用户实现的task封装为ftask，使用状态机和outcome管理ftask的执行过程。这些过程对用户是不可见的，直到用户调用get()方法。

>顺道明白了Callable实例是如何执行的，为什么实现Callable#call()方法时可以将受检异常抛到外层（而Runable#run()方法则必须在方法内处理，不能抛出）。

### get()

```java
public class FutureTask<V> implements RunnableFuture<V> {
...
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
...
}
```

* 5行利用定义状态的实际值判断ftask是否已完成，如果未完成（NEW、COMPLETING），则wait阻塞直到完成，该过程可抛出InterruptedException退出。
* 待ftask完成后，调用report()报告结束状态。

>5行的写法不可读，摒弃。

### report()

```java
public class FutureTask<V> implements RunnableFuture<V> {
...
    private V report(int s) throws ExecutionException {
        Object x = outcome;
        if (s == NORMAL)
            return (V)x;
        if (s >= CANCELLED)
            throw new CancellationException();
        throw new ExecutionException((Throwable)x);
    }
...
}
```

* 如果结束状态为NORMAL，则outcome保存了正常结果，泛型强转，返回。
* 7行利用定义状态的实际值判断ftask是否是被取消导致结束的（CANCELLED、INTERRUPTING、INTERRUPTED），如果是，则将抛出CancellationException。
* 如果不是被取消的，就是执行过程中task自己抛出了异常，则outcome保存了该异常t，包装返回ExecutionException。

>将异常t作为ExecutionException的cause包装起来，异常阅读方法参考[你真的会阅读Java的异常信息吗？](/2017/10/02/你真的会阅读Java的异常信息吗？/)。

#### CancellationException和ExecutionException

* CancellationException是非受检异常，原则上可以不处理，但仍然建议处理。
* ExecutionException是受检异常，在外层必须处理。

## 源码小结

* 实现Callable#.call()时可以将受检异常抛到外层。
* 不管实现Callable#.call()时是否抛出了受检异常，都要在FutureTask#get()时捕获ExecutionException；建议捕获CancellationException。
* FutureTask#get()中调用了阻塞方法，因此还需要捕获InterruptedException。
* CancellationException异常中不会给出取消原因，包括是否因为被中断。
* 工程上建议使用超时版的FutureTask#get()，超时会抛出TimeoutException，需要处理。

反观Future#get()的API声明：

```java
public interface Future<V> {
...
    /**
     * Waits if necessary for the computation to complete, and then
     * retrieves its result.
     *
     * @return the computed result
     * @throws CancellationException if the computation was cancelled
     * @throws ExecutionException if the computation threw an
     * exception
     * @throws InterruptedException if the current thread was interrupted
     * while waiting
     */
    V get() throws InterruptedException, ExecutionException;
...
}
```

right。

# 一种正确姿势

给出一种比较全面的正确姿势，仅供参考。

```java
int timeoutSec = 30;
try {
  MyResult result = ftask.get(timeoutSec, TimeUnit.SECONDS);
} catch (ExecutionException e) {
  Throwable t = e.getCause();
  // handle some checked exceptions
  if (t instantanceof IOExcaption) {
    xxx;
  } else if (...) {
    xxx;
  } else { // handle remained checked exceptions and unchecked exceptions
    throw new RuntimeException("xxx", t);
  }
} catch (CancellationException e) {
  xxx;
  throw new UnknownException(String.format("Task %s canceled unexpected", taskId));
} catch (TimeoutException e) {
  xxx;
  LOGGER.error(String.format("Timeout for %ds, trying to cancel task: %s", timeoutSec, taskId));
  ftask.cancel();
  LOGGER.debug(String.format("Succeed to cancel task: %s" % taskId));
} catch (InterruptedException e) {
  xxx;
}
```

* 根据实际需求删减。
* 猴子喜欢在一些语义模糊的地方加assert或抛出UnknownException代替注释。
* 对InterruptedException的处理暂时不讨论（少有的用于控制流程的异常，猴子理解的有点模糊），读者可参考[处理 InterruptedException](https://www.ibm.com/developerworks/cn/java/j-jtp05236.html)。

>换风格不错，写起来快多了。
