---
title: 你真的会阅读Java的异常信息吗？
tags:
  - Java
  - 原创
reward: true
date: 2017-10-02 14:19:56
---

给出如下异常信息：

```
java.lang.RuntimeException: level 2 exception
	at com.msh.demo.exceptionStack.Test.fun2(Test.java:17)
	at com.msh.demo.exceptionStack.Test.main(Test.java:24)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at com.intellij.rt.execution.application.AppMain.main(AppMain.java:147)
Caused by: java.io.IOException: level 1 exception
	at com.msh.demo.exceptionStack.Test.fun1(Test.java:10)
	at com.msh.demo.exceptionStack.Test.fun2(Test.java:15)
	... 6 more
```

学这么多年Java，你真的会阅读Java的异常信息吗？你能说清楚异常抛出过程中的事件顺序吗？

<!--more-->

>JDK版本：oracle java 1.8.0_102

# 需要内化的内容

## 写一个demo测试

上述异常信息在由一个demo产生：

```java
package com.msh.demo.exceptionStack;

import java.io.IOException;

/**
 * Created by monkeysayhi on 2017/10/1.
 */
public class Test {
  private void fun1() throws IOException {
    throw new IOException("level 1 exception");
  }

  private void fun2() {
    try {
      fun1();
    } catch (IOException e) {
        throw new RuntimeException("level 2 exception", e);
    }
  }


  public static void main(String[] args) {
    try {
      new Test().fun2();
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

>这次我复制了完整的文件内容，使文章中的代码行号和实际行号一一对应。

根据上述异常信息，异常抛出过程中的事件顺序是：

1. 在Test.java的第10行，抛出了一个IOExceotion("level 1 exception") e1
2. 异常e1被逐层向外抛出，直到在Test.java的第15行被捕获
3. 在Test.java的第17行，根据捕获的异常e1，抛出了一个RuntimeException("level 2 exception"， e1) e2
4. 异常e2被逐层向外抛出，直到在Test.java的第24行被捕获
5. 后续没有其他异常信息，经过必要的框架后，由程序自动或用户主动调用了e2.printStackTrace()方法

## 如何阅读异常信息

那么，如何阅读异常信息呢？有几点你需要认识清楚：

* **异常栈以FILO的顺序打印，位于打印内容最下方的异常最早被抛出，逐渐导致上方异常被抛出**。位于打印内容最上方的异常最晚被抛出，且没有再被捕获。从上到下数，第`i+1`个异常是第`i`个异常被抛出的原因`cause`，以“Caused by”开头。
* **异常栈中每个异常都由异常名+细节信息+路径组成**。异常名从行首开始（或紧随"Caused by"），紧接着是细节信息（为增强可读性，需要提供恰当的细节信息），从下一行开始，跳过一个制表符，就是路径中的一个位置，一行一个位置。
* **路径以FIFO的顺序打印，位于打印内容最上方的位置最早被该异常经过，逐层向外抛出**。最早经过的位置即是异常被抛出的位置，逆向debug时可从此处开始；后续位置一般是方法调用的入口，JVM捕获异常时可以从方法栈中得到。对于cause，其可打印的路径截止到被包装进下一个异常之前，之后打印“... 6 more”，表示cause作为被包装异常，在这之后还逐层向外经过了6个位置，但这些位置与包装异常的路径重复，所以在此处省略，而在包装异常的路径中打印。“... 6 more”的信息不重要，可以忽略。

现在，回过头再去阅读示例的异常信息，是不是相当简单？

为了帮助理解，我尽可能通俗易懂的描述了异常信息的结构和组成元素，可能会引入一些纰漏。阅读异常信息是Java程序猿的基本技能，希望你能内化它，忘掉这些冗长的描述。

>如果还不理解，建议你亲自追踪一次异常的创建和打印过程，使用示例代码即可，它很简单但足够。难点在于异常是JVM提供的机制，你需要了解JVM的实现；且底层调用了很多native方法，而追踪native代码没有那么方便。

# 扩展

## 为什么有时我在日志中只看到异常名"java.lang.NullPointerException"，却没有异常栈

示例的异常信息中，异常名、细节信息、路径三个元素都有，但是，**由于JVM的优化，细节信息和路径可能会被省略**。

这经常发生于服务器应用的日志中，由于相同异常已被打印多次，如果继续打印相同异常，JVM会省略掉细节信息和路径队列，**向前翻阅即可找到完整的异常信息**。

>猴哥之前使用Yarn的Timeline Server时遇到过该问题。你能体会那种感觉吗？卧槽，为什么只有异常名没有异常栈？没有异常栈怎么老子怎么知道哪里抛出的异常？线上服务老子又不能停，全靠日志了啊喂！

网上有不少相同的case，比如[NullPointerException丢失异常堆栈信息](http://blog.csdn.net/taotao4/article/details/43918131)，读者可以参照这个链接实验一下。

## 如何在异常类中添加成员变量

为了恰当的表达一个异常，我们有时候需要自定义异常，并添加一些成员变量，打印异常栈时，自动补充打印必要的信息。

追踪打印异常栈的代码：

```java
...
    public void printStackTrace() {
        printStackTrace(System.err);
    }
...
    public void printStackTrace(PrintStream s) {
        printStackTrace(new WrappedPrintStream(s));
    }
...
    private void printStackTrace(PrintStreamOrWriter s) {
        // Guard against malicious overrides of Throwable.equals by
        // using a Set with identity equality semantics.
        Set<Throwable> dejaVu =
            Collections.newSetFromMap(new IdentityHashMap<Throwable, Boolean>());
        dejaVu.add(this);

        synchronized (s.lock()) {
            // Print our stack trace
            s.println(this);
            StackTraceElement[] trace = getOurStackTrace();
            for (StackTraceElement traceElement : trace)
                s.println("\tat " + traceElement);

            // Print suppressed exceptions, if any
            for (Throwable se : getSuppressed())
                se.printEnclosedStackTrace(s, trace, SUPPRESSED_CAPTION, "\t", dejaVu);

            // Print cause, if any
            Throwable ourCause = getCause();
            if (ourCause != null)
                ourCause.printEnclosedStackTrace(s, trace, CAUSE_CAPTION, "", dejaVu);
        }
    }
...
```

暂不关心同步问题，可知，打印异常名和细节信息的代码为：

```java
            s.println(this);
```

JVM在运行期通过动态绑定实现this引用上的多态调用。继续追踪的话，最终会调用this实例的toString()方法。所有异常的最低公共祖先类是Throwable类，它提供了默认的toString()实现，大部分常见的异常类都没有覆写这个实现，我们自定义的异常也可以直接继承这个实现：

```java
...
    public String toString() {
        String s = getClass().getName();
        String message = getLocalizedMessage();
        return (message != null) ? (s + ": " + message) : s;
    }
...
    public String getLocalizedMessage() {
        return getMessage();
    }
...
    public String getMessage() {
        return detailMessage;
    }
...
```

显然，默认实现的打印格式就是示例的异常信息格式：异常名（全限定名）+细节信息。detailMessage由用户创建异常时设置，因此，如果有自定义的成员变量，我们通常在toString()方法中插入这个变量。参考`com.sun.javaws.exceptions`包中的`BadFieldException`，看看它如何插入自定义的成员变量field和value：

```java
  public String toString() {
    return this.getValue().equals("https")?"BadFieldException[ " + this.getRealMessage() + "]":"BadFieldException[ " + this.getField() + "," + this.getValue() + "]";
  }
```

>严格的说，`BadFieldException`的toString中并没有直接插入field成员变量。不过这不影响我们理解，感兴趣的读者可自行翻阅源码。

# 总结

根据异常信息debug是程序员的基本技能，这里围绕异常信息的阅读和打印过程作了初步探索，后续还会整理一下常用的异常类，结合[程序猿应该记住的几条基本规则](/2017/09/20/程序猿应该记住的几条基本规则/)，更好的理解如何用异常帮助我们写出clean code。

>Java相当完备的异常处理机制是一把双刃剑，用好它能增强代码的可读性和鲁棒性，用不好则会让代码变的更加不可控。例如，在空指针上调用成员方法，运行期会抛出异常，这是很自然的——但是，是不可控的等待它在某个时刻某个位置抛出异常（实际上还是“确定”的，但对于debug来说是“不确定”的），还是可控的在进入方法伊始就检查并主动抛出异常呢？进一步的，哪些异常应该被即刻处理，哪些应该继续抛到外层呢？抛往外层时，何时需要封装异常呢？看看String#toLowerCase()，看看ProcessBuilder#start()，体会一下。
