---
title: Java模拟赛跑过程
tags:
  - Java
  - 并发
  - 面试
  - 原创
reward: true
date: 2017-10-08 18:10:20
---

Java并发面试中的一个经典问题——手写代码模拟赛跑过程。该问题考查CountDownLatch的用法，比[Java实现生产者-消费者模型](/2017/10/08/Java实现生产者-消费者模型/)的考查更直接：

* 对Java并发模型的理解
* 对Java并发编程接口（CountDownLatch）的熟练程度
* **简化问题的能力**
* bug free
* coding style

<!--more-->

>JDK版本：oracle java 1.8.0_102

需要注意是“简化问题的能力”。建议读者在继续阅读之前，自己先实现一个版本，之后再与文中的代码比较，看哪种思路更简洁，实现更美观。

# 需求

问题很经典，则需求概括如下：

* 所有选手就位后，裁判鸣枪开始比赛
* 选手到达终点或中途退赛后，记录成绩
* 所有选手都已记录成绩后，结束比赛

再次建议读者自己先实现一个版本，再继续阅读。

# 设计 && 实现

## 设计

题目描述很清晰，现简化需求并设计如下：

```
选手逐渐就位 -> 开始比赛 -> 选手逐渐跑到终点或发生异常 -> 比赛结束
```

在三个连接处各使用一个CountDownLatch即可。

## 实现

三个连接处分别命名为ready、start、end，参赛者数量racerCnt，则：

```java
public class Race {

  public static void race(int racerCnt) throws InterruptedException {
    CountDownLatch ready = new CountDownLatch(racerCnt);
    CountDownLatch start = new CountDownLatch(1);
    CountDownLatch end = new CountDownLatch(racerCnt);

    for (int i = 0; i < racerCnt; i++) {
      final int racerNo = i;
      new Thread(new Runnable() {
        @Override
        public void run() {
          System.out.println("ready: " + racerNo);
          ready.countDown();
          try {
            start.await();
            System.out.println("start: " + racerNo);
          } catch (InterruptedException e) {
            System.out.println("I am hurt, and have to interrupt the race...");
            Thread.currentThread().interrupt();
          } finally {
            System.out.println("end: " + racerNo);
            end.countDown();
          }
        }
      }).start();
    }

    ready.await();
    System.out.println("********************** all ready!!! **********************");
    System.out.println("********************** will start soon **********************");
    start.countDown();
    end.await();
    System.out.println("********************** all end!!! **********************");
  }

  public static void main(String[] args) throws InterruptedException {
    race(5);
  }
}
```

选择一个比较乱的输出：

```
ready: 1
ready: 0
ready: 2
ready: 3
ready: 4
********************** all ready!!! **********************
********************** will start soon **********************
start: 1
start: 2
start: 3
end: 3
start: 0
end: 0
end: 2
end: 1
start: 4
end: 4
********************** all end!!! **********************
```

很明显：第一名是3号选手；最后一名是4号选手；3号跑完的时候，0号还没有开跑。

# 总结

用CountDownLatch模拟赛跑过程虽然简单，但能简洁、美观的实现却不容易，很考验面试者简化问题的能力。
