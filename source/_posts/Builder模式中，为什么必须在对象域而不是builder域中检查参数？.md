---
title: Builder模式中，为什么必须在对象域而不是builder域中检查参数？
tags:
  - Java
  - 设计模式
  - 原创
reward: true
date: 2017-09-20 18:50:50
---

>这个问题仅对于构造不可变类有意义，如果构造可变类，放在哪里都无法保障安全。

<!--more-->

使用Builder模式构建不可变类ImmutableObj的实例imObj时，应该把字段的检查放在对象域。可以从两个角度分析：

* 如果放在builder域（一般是Builder#build()方法中），会存在TOC攻击，即：时刻T1参数检查通过，时刻T2修改其中可变的参数，时刻T3调用构造器，则时刻T3时，参数又变得不合法了。
* 如果放在对象域（一般是类的构造方法中），还要**严格在完成所有字段的初始化之后（初始化之后就不可变了），再检查字段**，从而彻底避免TOC攻击。如下：

```java
public class ImmutableObj {
  private int[] nums;
  
  private ImmutableObj(int[] nums) {
    this.nums = nums.clone();
    // 完成所有字段的初始化后再检查字段
    checkArgs();
  }
  
  private void checkArgs() {
    if (...illegal...) {
      throw new IllegalArgumentException("Illegal nums ...: " + Arrays.toString(nums));
    }
  }
}
```

当然，上述表述有些绝对了：**如果参数本身就是不可变的，那么任何时候检查都是可以的**。

>PS：参数指Builder实例的成员变量，字段指ImmutableObj的成员变量。参数是相对于ImmutableObj的称法。
