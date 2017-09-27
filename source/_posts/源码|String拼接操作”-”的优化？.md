---
title: 源码|String拼接操作”+”的优化？
tags:
  - Java
  - JVM
  - 原创
reward: true
date: 2017-09-23 15:54:42
---

很多讲Java优化的文章都会强调对String拼接的优化。倒不用特意记，本质上在于对不可变类优势和劣势的理解上。

需要关注的是编译器对String拼接做出的优化，在简单场景下的性能能够与StringBuilder相当，复杂场景下仍然有较大的性能问题。网上关于这一问题讲的非常乱；如果我讲的有什么纰漏，也欢迎指正。

<!--more-->

>本文用到了反编译工具jad。在查阅网上关于String拼接操作的优化时发现了这个工具，能同时反编译出来源码和字节码，亲测好用，[点我下载](https://varaneckas.com/jad/)。

# String拼接的性能问题

优化之前，每次用”+”拼接，都会生成一个新的String。特别在循环拼接字符串的场景下，性能损失是极其严重的：

1. 空间浪费：每次拼接的结果都需要创建新的不可变类
2. 时间浪费：创建的新不可变类需要初始化；产生大量“短命”垃圾，影响 young gc甚至full gc

# 所谓简单场景

>简单场景和复杂场景是我乱起的名字，帮助理解编译器的优化方案。

简单场景可理解为在一句中完成拼接：

```java
int i = 0;
String sentence = “Hello” + “world” + String.valueOf(i) + “\n”;
System.out.println(sentence);
```

利用jad可看到优化结果：

```java
int i = 0;
String sentence = (new StringBuilder()).append(“Hello”).append(“world”).append(String.valueOf(i)).append(“\n”).toString();
System.out.println(sentence);
```

是不是很神奇，竟然把String的拼接操作优化成了StringBuilder#append()！

此时，可以认为已经将简单场景的空间性能、时间性能优化到最优（仅针对String拼接操作而言），看起来编译器已经完成了必要的优化。你可以测试一下，简单场景下的性能能够与StringBuilder相当。但是——“但是”以前的都是废话——编译器的优化对于复杂场景的帮助却很有限了。

# 所谓复杂场景

所谓复杂场景，可理解为“编译器不确定（或很难确定，于是不做分析）要进行多少次字符串拼接后才需要转换回String”。可能表述不准确，理解个大概就好。

我们分析一个最简单的复杂场景：

```java
String sentence = “”;
for (int i = 0; i < 10000000; i++) {
  sentence += “Hello” + “world” + String.valueOf(i) + “\n”;
}
System.out.println(sentence);
```

## 理想的优化方案

当然，无论什么场景，程序猿都可以手动优化：

* 在性能敏感的场景使用StringBuilder完成拼接。
* 在性能不敏感的场景使用更方便的String。

>PS：别吐槽，这样的API设计是合理的，**在合适的地方做合适的事**。

理想目标是把这件事交给javac和JIT：

* 设定一个拼接次数的阈值，超过阈值就启动优化（对于javac有一个编译期的阈值，JIT有一个运行期的阈值，以分阶段优化）。
* 优化时，在拼接前生成StringBuilder对象，将拼接操作换成StringBuilder#append()，继续使用该对象，直至“需要”String对象时，使用StringBuilder#toString()“懒加载”新的String对象。

该优化方案的难度在于代码分析：机器很难知道到底何时“需要”String对象，所以也很难在合适的位置注入代码完成“懒加载”。

虽然很难实现，但还是给出理想的优化结果，以供实际方案对比：

```java
String sentence = “”;
StringBuilder sentenceSB = new StringBuilder(sentence);
for (int i = 0; i < 10000000; i++) {
  sentenceSB.append(“Hello”).append(“world”).append(String.valueOf(i)).append(“\n”);
}
sentence = sentenceSB.toString();
System.out.println(sentence);
```

## 实际的优化方案

利用jad查看实际的优化结果：

```java
String sentence = “”;
for (int i = 0; i < 10000000; i++) {
  sentence = (new StringBuilder()).append(sentence).append(“Hello”).append(“world”).append(String.valueOf(i)).append(“\n”).toString();
}
System.out.println(sentence);
```

可以看到，实际上编译器的优化只能达到简单场景的最优：仅优化字符串拼接的一句。这种优化程度，对于上述复杂场景的性能提升很有限，循环时还是会生成大量短命垃圾，特别是字符串拼接到很大的时候，空间和时间上都是致命的。

通过对理想方案的分析，我们也能理解编译器优化的无奈之处：编译器无法（或很难）通过代码分析判断何时是最晚进行懒加载的时机。为什么呢？我们将代码换个形式可能更容易理解：

```java
String sentence = “”;
for (int i = 0; i < 10000000; i++) {
  sentence = sentence + “Hello” + “world” + String.valueOf(i) + “\n”;
}
System.out.println(sentence);
```

观察第3行的代码，等式右侧引用了sentence。我肉眼知道这句话只完成了字符串拼接，机器呢？最起码，现在的机器还很难通过代码判断。

>待以后将人工智能与编译优化结合起来，就算只能以90%的概率完成优化，也是非常cool的。

# 总结

这个问题我没有做性能测试。其实也没必要过于深究，与其让编译器以隐晦的方式完成优化，不如用代码进行主动、清晰的优化，让代码能够“自解释”。

那么，如果需要优化，使用StringBuilder吧。
