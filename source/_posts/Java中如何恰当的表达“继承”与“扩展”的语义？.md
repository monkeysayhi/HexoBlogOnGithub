---
title: Java中如何恰当的表达“继承”与“扩展”的语义？
tags:
  - Java
  - 猿格
  - 原创
reward: true
date: 2017-09-20 21:08:23
---

”继承“是Java的面向对象学习过程中的大难题，原因有二：

* ”is-A“的关系本身就不好理解
* Java中的extends“扩展”与面向对象中的“继承”inheritance不是一一对应的。

很多书里认为继承与扩展是一一对应的，但个人不这样认为。并且以我的观点，能更好的指导开发工作如何进行继承与扩展的程序设计和编码实现。本文尝试以通俗的语言陈述Java中如何恰当的表达“继承”与“扩展”的语义。

<!--more-->

>JDK版本：oracle java 1.8.0_102

# 继承和扩展的区别

>继承是面向对象中的语义，表示“is—A”的关系。而扩展是Java语言中的语义，表示“从一个雷扩展出另一个类”，两个类之间不一定具有“is—A”的关系。

单纯从概念上陈述继承和扩展的区别可能难以理解。我们尝试以面向对象中的继承语义为参照标准，假设Java中的扩展方式也可以表示继承，那么：

* **类继承（implementation inheritance）是不恰当的**。也就是说，用extends关键字在类之间表达的所谓“继承”（简称“A类继承”）不等于面向对象中的继承，不恰当。
* **接口继承（interface inheritance）是恰当的**，这分为两种情况：
  * 用extends关键字在接口之间表达“继承”（简称“B类继承”）是恰当的。
  * 用implements关键字在接口与类之间表达“继承”（简称“C类继承”）是恰当的。

>PS：如果一种实现方式能够表达继承的语义，则称这种实现方式是恰当的。

B类继承和C类继承实现的是面向对象中的“继承”语义，自然恰当，无需解释。而A类继承实现的是Java语言中的“扩展”语义，即“用一个类扩展另一个类”，主要作用是简化代码的编写过程。

**误区产生在A类继承上，它表达的是扩展的，但我们总把它当做继承来用**。

>如果现在就明白了继承与扩展的区别，你可以关闭文章了；如果还觉得有些糊涂，我们再用几个例子深入讲解扩展和A类继承。

# 扩展和A类继承

仍旧以继承的语义为参照标准。

## 为什么A类继承是不恰当的

A类继承可以且仅可以表达“扩展”的语义，这是Java层的语义，与面向对象无关。

假设我们希望自己实现一个HashSet，使它能够记录下总共添加过多少个元素，用A类继承实现：

```java
public class RecordedHashSet1<E> extends HashSet<E> {
  private int addOpCnt = 0;

  public RecordedHashSet1() {
    super();
  }

  @Override
  public boolean add(E e) {
    addOpCnt++;
    return super.add(e);
  }

  // 继承破坏了父类的封装性
  @Override
  public boolean addAll(Collection<? extends E> c) {
    addOpCnt += c.size();
    return super.addAll(c);
  }

  public int getAddOpCnt() {
    return addOpCnt;
  }

  public static void main(String[] args) {
    RecordedHashSet1<Integer> set = new RecordedHashSet1<>();
    set.addAll(Arrays.asList(1, 2, 3, 4, 5, 6));
    System.out.println(set.getAddOpCnt());
  }
}
```

运行上述代码，猜猜会输出什么？

你可能以为是6。如果是这样，那么A类继承在这个例子上就是恰当的。

*程序会输出12，但是很明显我们总共只添加过6次元素*。在我们的设想中，RecordedHashSet1 “is a” HashSet，因此RecordedHashSet1的行为应该与HashSet保持一致。现在集合只添加过6次元素，记录添加次数的成员变量addOpCnt却等于12，行为不一致。

>实际上，RecordedHashSet1 “is not a” HashSet，它仅是一个Set，扩展了HashSet。后面我们会看到更详细的解释。

为什么会产生这种奇观的行为呢？我们需要了解父类addAll()方法的实现：

```java
    // addAll 继承自 AbstractCollection
    public boolean addAll(Collection<? extends E> c) {
        boolean modified = false;
        for (E e : c)
            if (add(e))
                modified = true;
        return modified;
    }
```

在addAll()方法中，通过add()方法将每个元素添加到集合。所以addAll 6个元素时，RecordedHashSet1#addAll()会先将计数+6，再调用super.addAll()，而super.addAll()内部会再将计数+6。

## 小结

RecordedHashSet1覆写add()与addAll()的选择是正确的，错的是我们选择了A类继承。**如果想用A类继承表达面向对象中的“继承”，那么子类必须了解父类的实现，破坏了父类的封装性，与继承语义不兼容**。所以，A类继承能且仅能表达“扩展”，无法恰当的表达“继承”

# 如何同时恰当的表达扩展和继承的语义

A类继承只能表达扩展，好处是编码简单，坏处是破坏父类的封装性，从而无法表达继承的语义；B、C类继承只能表达继承。

使用接口+组合（或称“复合”，composition）（简称“D类继承”），可以同时表达子类与父类间的扩展关系，和子类与接口间的继承关系。

## 用D类继承改写上述需求

D类继承常被称为开闭原则：

>面向组合开放，面向继承关闭。

在开闭原则中，对象的类型用接口定义，接口与接口之间符合B类继承，接口与子类之间符合C类继承。那么如何表达扩展类与被扩展类之间的扩展关系呢？

还是上面那个需求，用D类继承实现：

```java
// 偷懒使用了Java7的JDK
public class RecordedHashSet2<E>
    extends AbstractSet<E>
    implements Set<E> {
  private Set<E> set = new HashSet<E>();
  private int addOpCnt = 0;

  public RecordedHashSet2() {
    super();
  }

  @Override
  public Iterator<E> iterator() {
    return set.iterator();
  }

  @Override
  public int size() {
    return set.size();
  }

  @Override
  public boolean add(E e) {
    addOpCnt++;
    return set.add(e);
  }

  @Override
  public boolean addAll(Collection<? extends E> c) {
    addOpCnt += c.size();
    return set.addAll(c);
  }

  public int getAddOpCnt() {
    return addOpCnt;
  }

  public static void main(String[] args) {
    RecordedHashSet2<Integer> set = new RecordedHashSet2<>();
    set.addAll(Arrays.asList(1, 2, 3, 4, 5, 6));
    System.out.println(set.getAddOpCnt());
  }
}
```

再次运行上述代码，正确输出6。

本质上，RecordedHashSet2 “is not a” HashSet，它仅仅是一个Set或RecordedSet，需要表达的是RecordedHashSet2类与Set接口之间的继承关系；同时，RecordedHashSet2增加了HashSet的功能，需要表达RecordedHashSet2与HashSet之间的扩展关系。D类继承恰到好处。

## 小结

D类继承能同时恰当的表达扩展和继承的语义。但与A类继承相比，D类继承加重了编码的负担：代码长，需要将每一个请求都转发给转发类处理。

# 总结

到这里，希望你能明白扩展与继承的区别。现在回答题目的问题，Java中如何恰当的表达“继承”与“扩展”的语义？

* 仅表达继承用B类继承或C类继承
* 仅表达扩展用A类继承或简化的D类继承（不继承接口）
* 同时表达继承和扩展用D类继承

>PS：要真正理解extends、implements与继承的关系。extends从一个类扩展出另一个类，implements从一个接口实现一个类或抽象类，它们作为Java中的关键字，语义本身是正确的；只是与面向对象中的继承无法一一对应。
