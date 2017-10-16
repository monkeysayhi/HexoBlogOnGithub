---
title: 【刷题】Linked List Cycle II
tags: 
  - LintCode
  - 算法
  - 面试
reward: true
date: 2017-08-19 21:38:40
---

[原题戳我](http://www.lintcode.com/en/problem/linked-list-cycle-ii)

<!--more-->

# 题目

## Description

Given a linked list, return the node where the cycle begins.

If there is no cycle, return null.

## Example
Given -21->10->4->5, tail connects to node index 1，return 10

## Challenge 

Follow up:

Can you solve it without using extra space?

# 分析

## 问题分解

### basic problem

>单链表中确定有无环

暴力方法是使用hash表统计节点的出现次数，O(n)的space。不过一般是希望以O(1)的space解决：

1. 设置一个快指针fast和一个慢指针slow，它们同时从链表头开始往后遍历，快指针每次移动两个位置，慢指针每次移动一个位置
2. 如果在快指针访问到null（即无环）之前，快指针和慢指针相遇，就说明有环

### follow up

>确定环入口入口的位置

和basic problem一样，可以用hash表解决。而follow up的目的仍然是希望O(1)的space解决：

1. 重复上述判定有环无环的过程
2. 用一个新指针指向head，与slow指针同时一次移动一个位置
3. 当head与slow相遇时，指针所指的节点即入口节点

## 证明

### 相遇一定有环

反证法：

如果无环，则fast一定比slow先到达null，不会相遇。因此，如果fast与slow相遇，则一定有环。

得证。

### 第一次相遇一定发生在slow第一次入环的过程中

**该结论是下一步推导的前提**。

设链表头节点为head，环入口节点为entrance，head到entrance共有n个节点，环上共有m个节点。显然，fast和slow相遇的节点一定在环中，设从entrance到这个节点共k个节点。

>PS：代码实现时，需要关注m、n、k等计算时是否包含head、entrance等，但证明时无需关心，仅仅是加减一个常数项的事情。

**fast的速度是slow的两倍**。则，fast第一次到entrance时，slow到达链表中部节点mid，设head到mid共有s个节点，则此时，slow走了s个节点，fast走了2s个节点。我们让slow再走s个节点，fast再走2s个节点，分情况讨论：

#### 如果entrance等于head

此情况下环最长。经过s个节点，slow恰好第一次到达entrance；经过2s个节点，fast恰好第二次到达entrance。在此过程中，slow有且仅有一次从mid走到entrance，fast有且仅有一次从head经过mid走到entrance，从而，fast与slow必然有且仅有一次在mid于entrance之间相遇，这是二者第一次相遇。由于相遇点一定在环中，因此*第一次相遇一定发生在slow第一次入环的过程中*。

#### 如果entrance不等于head

此情况下，环均比第一种情况短。重复上述过程，slow仍然有且仅有一次从mid走到entrance，但fast却由于环的缩短，可能不止一次与slow相遇并走到entrance。我们只关注第一次相遇，显然，*第一次相遇仍然发生在slow第一次入环的过程中*。

得证。

### 如何寻找入口节点

我们现在知道，**“第一次相遇一定发生在slow第一次入环的过程中”**，那么此时slow共走了`n+k`个节点，fast共移动了`n+k + x*m`个节点。fast速度是slow的2倍，则有`2*(n+k) = n+k + x*m`，其中x表示fast已经在环中走的圈数。由此可得，`n+k = x*m`，从而`n = (x-1)*m + m-k`(x>=1, m>=k)。

在有环链表中，我们只能基于移动和相遇进行判断。basic problem中，我们希望通过相遇判断是否有环，follow up中，可以试着用相遇找到entrance节点。

观察式`n = (x-1)*m + m-k`：

* `n`为head到entrance的节点数
* `m`为环的长度
* `m-k`为从fast、slow的相遇点到entrance的节点数

如果让slow继续走`m-k`个节点，此时slow将恰好位于entrance；再循环`y = x-1`圈，仍然处于entrance。增加一个新的指针`p = head`，则可以这样理解上式：**使p和slow同时开始移动（slow从刚才的相遇点开始），都一次移动一个位置，则当p第一次经过n个节点走到entrance时，slow恰好先经过了`m-k`个节点，再走了整y圈回到entrance**。

此时，p与slow相遇，相遇点即为entrance。

# 代码

```java
/**
 * Definition for ListNode.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int val) {
 *         this.val = val;
 *         this.next = null;
 *     }
 * }
 */ 
public class Solution {
    /**
     * @param head: The first node of linked list.
     * @return: The node where the cycle begins. 
     *           if there is no cycle, return null
     */
    public ListNode detectCycle(ListNode head) {  
        // write your code here
        if (head == null) {
            return null;
        }
        
        ListNode slow = head;
        ListNode fast = head;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
            if (slow == fast) {
                break;
            }
        }
        if (fast == null || fast.next == null) {
            return null;
        }
        
        ListNode p = head;
        while (p != slow) {
            p = p.next;
            slow = slow.next;
        }
        return p;
    }
}
```
