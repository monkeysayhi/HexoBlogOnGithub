---
title: 源码|批量执行invokeAll()&&多选一invokeAny()
tags:
  - Java
  - 并发
  - 原创
reward: true
date: 2017-11-19 21:40:13
---


ExecutorService中定义了两个批量执行任务的方法，invokeAll()和invokeAny()，在批量执行或多选一的业务场景中非常方便。invokeAll()在所有任务都完成（包括成功/被中断/超时）后才会返回，invokeAny()在任意一个任务成功（或ExecutorService被中断/超时）后就会返回。

AbstractExecutorService实现了这两个方法，本文将先后分析invokeAll()和invokeAny()两个方法的源码实现。

<!--more-->

# invokeAll()

invokeAll()在所有任务都完成（包括成功/被中断/超时）后才会返回。有不限时和限时版本，从更简单的不限时版入手。

## 不限时版

```java
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    throws InterruptedException {
    if (tasks == null)
        throw new NullPointerException();
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
    boolean done = false;
    try {
        for (Callable<T> t : tasks) {
            RunnableFuture<T> f = newTaskFor(t);
            futures.add(f);
            execute(f);
        }
        for (int i = 0, size = futures.size(); i < size; i++) {
            Future<T> f = futures.get(i);
            if (!f.isDone()) {
                try {
                    f.get(); // 无所谓先执行哪个任务的get()方法
                } catch (CancellationException ignore) {
                } catch (ExecutionException ignore) {
                }
            }
        }
        done = true;
        return futures;
    } finally {
        if (!done)
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
    }
}
```

8-12行，先将所有任务都提交到线程池（当然，任何ExecutorService均可）中。

>严格来说，不是“提交”，而是“执行”。执行可能是同步或异步的，取决于线程池的策略。不过由于我们仅讨论异步情况（同步同理），用“提交”一词更容易理解。下同。

13-22行，for循环的目的是阻塞调用invokeAll的线程，直到所有任务都执行完毕。当然我们也可以使用其他方式实现阻塞，不过这种方式是最简单的：

* 15行如果f.isDone()返回true，则当前任务已结束，继续检查下一个任务；否则，调用f.get()让线程阻塞，直到当前任务结束。
* 17行**无所谓先执行哪一个FutureTask实例的get()方法**。由于所有任务并发执行，总体阻塞时间取决于于是耗时最长的任务，从而实现了invodeAll的阻塞调用。
* 18-20行没有捕获InterruptedException。如果有任务被中断，主线程将抛出InterruptedException，以响应中断。

最后，为防止在全部任务结束之前过早退出，23行、25-29行相配合，如果done不为true（未执行到40行就退出了）则取消全部任务。

## 限时版

```java
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                     long timeout, TimeUnit unit)
    throws InterruptedException {
    if (tasks == null)
        throw new NullPointerException();
    long nanos = unit.toNanos(timeout);
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
    boolean done = false;
    try {
        for (Callable<T> t : tasks)
            futures.add(newTaskFor(t));

        final long deadline = System.nanoTime() + nanos;
        final int size = futures.size();

        // Interleave time checks and calls to execute in case
        // executor doesn't have any/much parallelism.
        for (int i = 0; i < size; i++) {
            execute((Runnable)futures.get(i));
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) // 及时检查是否超时
                return futures;
        }

        for (int i = 0; i < size; i++) {
            Future<T> f = futures.get(i);
            if (!f.isDone()) {
                if (nanos <= 0L) // 及时检查是否超时
                    return futures;
                try {
                    f.get(nanos, TimeUnit.NANOSECONDS);
                } catch (CancellationException ignore) {
                } catch (ExecutionException ignore) {
                } catch (TimeoutException toe) {
                    return futures;
                }
                nanos = deadline - System.nanoTime();
            }
        }
        done = true;
        return futures;
    } finally {
        if (!done)
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
    }
}
```

10-11行，先将所有任务封装为FutureTask，添加到futures列表中。

18-23行，每提交一个任务，就立刻判断是否超时。这样的话，如果在任务全部提交到线程池中之前，就已经达到了超时时间，则能够尽快检查出超时，结束提交并退出。

>对于限时版，将封装任务与提交任务拆开是必要的。

28-29行，每次在调用限时版f.get()进入阻塞状态之前，先检查是否超时。这里也是希望超时后，能够尽快发现并退出。

其他同不限时版。

# invokeAny()

invokeAny()在任意一个任务成功（或ExecutorService被中断/超时）后就会返回。也分为不限时和限时版本，但为了进一步保障性能，invokeAny()的实现思路与invokeAll()略有不同。

```java
public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
    throws InterruptedException, ExecutionException {
    try {
        return doInvokeAny(tasks, false, 0);
    } catch (TimeoutException cannotHappen) {
        assert false;
        return null;
    }
}
public <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                       long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    return doInvokeAny(tasks, true, unit.toNanos(timeout));
}
```

内部调用了doInvokeAny()。

>学习5-8行的写法，代码自解释。

## doInvokeAny()

简化如下：

```java
private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,
                          boolean timed, long nanos)
    throws InterruptedException, ExecutionException, TimeoutException {
...
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks);
    ExecutorCompletionService<T> ecs =
        new ExecutorCompletionService<T>(this);
...
    try {
        ExecutionException ee = null;
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        Iterator<? extends Callable<T>> it = tasks.iterator();

        futures.add(ecs.submit(it.next()));
        --ntasks;
        int active = 1;

        for (;;) {
            Future<T> f = ecs.poll();
            if (f == null) {
                if (ntasks > 0) {
                    --ntasks;
                    futures.add(ecs.submit(it.next()));
                    ++active;
                }
                else if (active == 0)
                    break;
                else if (timed) {
                    f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                    if (f == null)
                        throw new TimeoutException();
                    nanos = deadline - System.nanoTime();
                }
                else
                    f = ecs.take();
            }
            if (f != null) {
                --active;
                try {
                    return f.get();
                } catch (...) {
                    ee = ...;
                }
            }
        }
...
        throw ee;
    } finally {
        for (int i = 0, size = futures.size(); i < size; i++)
            futures.get(i).cancel(true);
    }
}
```

要点：

* ntasks维护未提交的任务数，active维护已提交未结束的任务数。
* 内部使用ExecutorCompletionService维护已完成的任务。
* 如果没有任务成功结束，则返回捕获的最后一个异常。
* 第一个任务是必将被执行的，其他任务按照迭代器顺序增量提交。

14行先向线程池提交一个任务（迭代器第一个），ntasks--，active=1：

```java
futures.add(ecs.submit(it.next()));
--ntasks;
int active = 1;
```

>这里是真“提交”了，不是“执行”。

然后18-45行循环检查是否有任务成功结束。

首先，19行通过及时返回的poll()方法，尝试取出一个已完成的任务：

```java
Future<T> f = ecs.poll();
```

根据f的结果，分成两种情况讨论。

>ExecutorCompletionService默认使用LinkedBlockingQueue作为任务队列。对LinkedBlockingQueue不熟悉的可参照[源码|并发一枝花之BlockingQueue](/2017/10/18/源码|并发一枝花之BlockingQueue/)。

### case1：如果有任务完成

如果有任务完成，则f不为null，进入40-49行，active--，并尝试取出任务结果：

```java
if (f != null) {
    --active;
    try {
        return f.get();
    } catch (...) {
        ee = ...;
    }
}
```

* 如果能够成功取出，即当前任务已成功结束，直接返回。
* 如果抛出异常，则当前任务异常结束，使用ee记录异常。

显然，如果已完成的任务是异常结束的，invokeAny()不会退出，而是继续查看其它任务。

>FutureTask#get()的用法参照[源码|使用FutureTask的正确姿势](/2017/10/29/源码|使用FutureTask的正确姿势/)。

### case2：如果没有任务完成

如果没有任务完成，则f为null，进入23-39行，判断是继续提交任务、退出还是等待任务结果：

```java
if (f == null) {
    if (ntasks > 0) { // check1
        --ntasks;
        futures.add(ecs.submit(it.next()));
        ++active;
    }
    else if (active == 0) // check2
        break;
    else if (timed) { // check3
        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
        if (f == null)
            throw new TimeoutException();
        nanos = deadline - System.nanoTime();
    }
    else // check4
        f = ecs.take();
}
```

* check1：如果还有剩余任务（ntasks > 0），那就继续提交，同时ntasks--，active++。
* check2：如果没有剩余任务了，且也没有已提交未结束的任务（active == 0），则表示全部任务均已执行结束，但没有一个任务是成功的，可以退出循环。退出循环后，将在47行抛出ee记录的最后一个异常。
* check3：如果可以没有剩余任务，但还有已提交未结束的任务，且开启了超时机制，则尝试使用超时版poll()等待任务完成。但是，如果这种情况下超时了，就表示整个invokeAny()方法超时了，所以poll()返回null的时候，要主动抛出TimeoutException。
* check4：如果可以没有剩余任务，但还有已提交未结束的任务，且未开启超时机制，则使用无限阻塞的take()方法，等待任务完成。

>这种一堆if-else的代码很丑。可修改如下：
>
```java
if (f == null) { // check1
    if (ntasks > 0) {
        --ntasks;
        futures.add(ecs.submit(it.next()));
        ++active;
        continue;
    }
    if (active == 0) { // check2
        assert ntasks == 0; // 防止自己改吧改吧把它这句判断挪到了前面
        break;
    }
    if (timed) { // check3
        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
        if (f == null) {
            throw new TimeoutException();
        }
        nanos = deadline - System.nanoTime();
    } else { // check4
        f = ecs.take();
    }
}
```
>
>修改依据：
>
>* check1、check2、check3/check4没有并列的判断关系
>* check3、check4有并列的判断关系，非此即彼
>* 结构更清爽
>

# 总结

不会写总结。。。

>但是会写吐槽啊！！！
>
>猴子现在每次写博客都经历着从“卧槽似乎很简单啊，写个毛”到“卧槽这跟想象的不一样啊！卧槽巨帅！”的心态崩塌，各位巨巨写的代码是真好看，性能还棒棒哒，羡慕崇拜打鸡血。哎，，，保持谦卑，并羡慕脸遥拜各位巨巨。
