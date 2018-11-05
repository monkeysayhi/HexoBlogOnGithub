---
title: Java中的常见OOM及原因
tags:
  - Java
  - JVM
  - 面试
  - 原创
reward: true
date: 2018-11-05 18:41:44
---

Java中的OOM（Out of Memory）指java.lang.OutOfMemoryError错误。了解JVM的基本原理后，很容易理解以下几种常见的OOM。

<!--more-->

# java.lang.OutOfMemoryError:Java heap space

这是最常见的OOM原因。

堆中主要存放各种对象实例，还有常量池等结构。当JVM发现堆中没有足够的空间分配给新对象时，抛出该异常。具体来讲，在刚发现空间不足时，会先进行一次Full GC，如果GC后还是空间不足，再抛出异常。

引起空间不足的原因主要有：

* 业务高峰，创建对象过多
* 内存泄露
* 内存碎片严重，无法分配给大对象

# java.lang.OutOfMemoryError:Metaspace

方法区主要存储类的元信息，实现在元数据区。当JVM发现元数据区没有足够的空间分配给加载的类时，抛出该异常。

引起元数据区空间不足的原因主要有：

* 加载的类太多，常见于Tomcat等容器中

但是元数据区被实现在堆外，主要受到进程本身的内存限制，这种实现下很难溢出。

## java.lang.OutOfMemoryError:Permgen space

jdk7中，方法区被实现在永久代中，错误原因同上。

永久代非常小，而且不会被回收，很容易溢出，因此，jdk8彻底废除了永久代，将方法区实现在元数据区。

# java.lang.OutOfMemoryError:Unable to create new native thread

以Linux系统为例，JVM创建的线程与操作系统中的线程一一对应，受到以下限制：

* 进程和操作系统的内存资源限制。其中，一个JVM线程至少要占用OS的线程栈+JVM的虚拟机栈 = 8MB + 1MB = 9MB（当然JVM实现可以选择不使用这1MB的JVM虚拟机栈）。
* 进程和操作系统的线程数限制。
* Linux中的线程被实现为轻量级进程，因此，还受到pid数量的限制。

当无法在操作系统中继续创建线程时，抛出上述异常。

解决办法从原因中找：

* 内存资源：调小OS的线程栈、JVM的虚拟机栈。
* 线程数：增大线程数限制。
* pid：增大pid范围。

# 其他异常

## java.lang.OutOfMemoryError:GC overhead limit exceeded

默认配置下，如果GC花费了98%的时间，回收的内存都不足2%的话，抛出该异常。

## java.lang.OutOfMemoryError:Out of swap space

如果JVM申请的内存大于可用物理内存，操作系统会将内存中的数据交换到磁盘上去（交换区）。如果交换区空间不足，抛出该异常。

---

>参考:
>
>* [Java内存溢出(OOM)异常完全指南](https://www.jianshu.com/p/2fdee831ed03)
>
