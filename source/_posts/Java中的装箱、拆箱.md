---
title: Java中的装箱、拆箱
tags:
  - Java
  - 面试
  - 原创
reward: true
date: 2018-11-13 11:04:12
---

* 装箱：将基本数据类型转换为包装器类型
* 拆箱：将包装器类型转换为基本数据类型

装箱与拆箱的过程时自动进行的，因此称为“自动装箱”、“自动拆箱”，属于编译期的语法糖。

<!--more-->

以基本类型int与包装类型Integer为例讨论。既然是编译期的语法糖，那么直接分析编译出来的字节码即可，可以使用jad工具（估计内部封装了javap一类的工具）。

# 装箱的实现

```java
  public static void main(String[] args) {
    Integer boxing = 1;
  }
```

jad解析class文件，发现根据字节码反编译为`Integer.valueOf(int)`方法：

```java
  public static void main(String args[]) {
    Integer boxing = Integer.valueOf(1);
    //    0    0:iconst_1        
    //    1    1:invokestatic    #2   <Method Integer Integer.valueOf(int)>
    //    2    4:astore_1        
    //    3    5:return          
  }
```

这是一个静态工厂方法，内部最终调用了Integer类的构造器`Integer.<init>(init)`：

```java
    public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }
```

特别的，Integer类默认缓存`-128~127`的包装对象（上界可通过`java.lang.Integer.IntegerCache.high`参数配置，最小127）。因此，缓存范围内的包装对象来自缓存的常量池，相同value包装对象的引用相等；缓存范围外的包装对象的引用均不相等。

# 拆箱的实现

```java
  public static void main(String[] args) {
    int unboxing = new Integer("2");
  }
```

拆箱的时候编译为`Integer.intValue()`方法：

```java
  public static void main(String args[]) {
    int unboxing = (new Integer("2")).intValue();
    //    0    0:new             #2   <Class Integer>
    //    1    3:dup             
    //    2    4:ldc1            #3   <String "2">
    //    3    6:invokespecial   #4   <Method void Integer(String)>
    //    4    9:invokevirtual   #5   <Method int Integer.intValue()>
    //    5   12:istore_1        
    //    6   13:return          
  }
```

该方法直接返回内部值`Integer#value`:

```java
    public int intValue() {
        return value;
    }
```

# 自动装箱与自动拆箱的陷阱

陷阱主要出现在包装类型与基本类型混用的场景中。

以“`==`”为例：

* 如果两个操作数都是基本类型，则使用基本类型的比较。
* 如果一个操作数是基本类型，另一个是包装类型，则拆箱为基本类型再比较：如果包装对象为null，抛出NullPointerException。（想想看，如果装箱为包装类型，显然无法发现null）
* 如果两个操作数都是包装类型，则比较二者的引用是否相等：如果二者内部值相等，但在缓存范围外，则必然不等。

>因此，如果要比较包装类型是否相等，最好显示的使用equals方法。

另外，如果是算术运算、除“`==`”外的逻辑运算、位运算等，则统一拆箱为基本类型再运算；有抛出NullPointerException的危险。
