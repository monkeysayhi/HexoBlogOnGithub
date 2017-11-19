---
title: 源码|newTaskFor()和适配器模式
tags:
  - Java
  - 并发
  - 设计模式
  - 面试
  - 原创
reward: true
date: 2017-11-19 22:11:12
---

AbstractExecutorService提供了一个创建任务的工厂方法——newTaskFor()。工厂方法大家很熟悉了，但newTaskFor()中用到的适配器模式却少有人提到。

<!--more-->

>高能预警！这篇，又！短！又！没！用！

# newTaskFor()和工厂方法模式

在AbstractExecutorService中，可以提交三种形式的task：

```java
public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<Void> ftask = newTaskFor(task, null);
    execute(ftask);
    return ftask;
}
public <T> Future<T> submit(Runnable task, T result) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task, result);
    execute(ftask);
    return ftask;
}
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

三种形式类似，主要区别在于工厂方法AbstractExecutorService#newTaskFor()：

```java
protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
    return new FutureTask<T>(runnable, value);
}
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
```

直接调用了FutureTask的有参构造函数。

FutureTask实现了RunnableFuture接口：

```java
public class FutureTask<V> implements RunnableFuture<V> 
```

该接口继承了Runnable、Future接口，并只有一个run方法。看下FutureTask的构造方法：

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

8行是一个适配器模式：

```java
this.callable = Executors.callable(runnable, result);
```

通过静态工厂方法兼适配器Executors.callable()将Runnbale实例和result适配为Callable实例。

# callable()和适配器模式

```java
public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
        throw new NullPointerException();
    return new RunnableAdapter<T>(task, result);
}
```

RunnableAdapter完成实际的适配工作：

```java
static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}
```

通过RunnableAdapter实例调用Callable#call()方法时，RunnableAdapter仅仅先调用Runnable#run()，再返回result。框架并没有什么地方会操作result——换句话说，**这个适配器仅仅是让Runable实例能够被提交到ExecutorService中，与返回值并没有半分钱的关系**。

>对，就是这么没用，但是用来讲适配器还是阔以的。

# 总结

通过AbstractExecutorService#newTaskFor()学习工厂方法模式，通过Executors.callable()学习适配器模式——嗯，，，虽然这个适配器没什么用。

>另外，注意到AbstractExecutorService#newTaskFor()的访问权限为protected，我们可以在扩展类中覆写或直接使用该方法。如ForkJoinPool：
>
>```java
>protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
>    return new ForkJoinTask.AdaptedRunnable<T>(runnable, value);
>}
>protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
>    return new ForkJoinTask.AdaptedCallable<T>(callable);
>}
>```
>
>适配器走起。
