---
title: 【刷题】Search in a Big Sorted Array
tags: 
  - LintCode
  - 算法
  - 面试
  - 原创
reward: true
date: 2017-08-04 13:25:33
---

[原题戳我](http://www.lintcode.com/en/problem/search-in-a-big-sorted-array/)。

<!--more-->

# 题目

## Description

Given a big sorted array with positive integers sorted by ascending order. The array is so big so that you can not get the length of the whole array directly, and you can only access the kth number by `ArrayReader.get(k)` (or `ArrayReader->get(k)` for `C++`). Find the first index of a target number. Your algorithm should be in O(log k), where k is the first index of the target number.

Return -1, if the number doesn't exist in the array.

>Notice:
If you accessed an inaccessible index (outside of the array), ArrayReader.get will return `2147483647`.

## Example

* Given [1, 3, 6, 9, 21, ...], and target = 3, return 1.
* Given [1, 3, 6, 9, 21, ...], and target = 4, return -1.

## Challenge 

O(log k), k is the first index of the given target number.

## Tags 

`Sorted Array` `Binary Search`

# 分析

这道题我是看题解才做出来的，思路非常好，有两个重点：

1. 可以O(logk)的时间缩小二分法的范围
2. 从而，可以将二分的最坏时间优化到O(logk)

第二点无需证明，下面讲解第一点。

## 以O(logk)的时间缩小二分法的范围

如果从0遍历到k，那么明显时间复杂度为O(k)，超过了了O(logk)。

要记得，我们的目的是确定一个数组的上界r，使O(r)=O(k)，继而在这段数组上进行二分查找，复杂度为O(logk)。因此，我们只需要将在O(logk)的时间内找到该r。

r的要求如下：

1. 满足O(r)=O(k)
2. 计算r的时间为O(logk)

即，寻找一个运算，进行O(logk)次，结果为O(k)。于是想到了**乘幂**：

```
2 ** O(logk) = O(k)
```

代码如下：

```java
    private int[] computeRange(ArrayReader reader, int target){
        int r = 1;
        while (reader.get(r) < target) {
            r <<= 1;
        }
        
        int l = r >> 1;
        while (r >= l && reader.get(r) == 2147483647) {
            r--;
        }
        if (r < l) {
            return null;
        }
        
        return new int[]{l, r};
    }
```

# 完整代码

```java
/**
 * Definition of ArrayReader:
 * 
 * class ArrayReader {
 *      // get the number at index, return -1 if index is less than zero.
 *      public int get(int index);
 * }
 */
public class Solution {
    /**
     * @param reader: An instance of ArrayReader.
     * @param target: An integer
     * @return : An integer which is the index of the target number
     */
    public int searchBigSortedArray(ArrayReader reader, int target) {
        // write your code here
        if (reader == null) {
            return -1;
        }
        
        if (reader.get(0) > target) {
            return -1;
        }
        
        int[] range = computeRange(reader, target);
        if (range == null) {
            return -1;
        }
        
        int k = bsearchLowerBound(reader, range[0], range[1], target);
        if (reader.get(k) != target) {
            return -1;
        }
        
        return k;
    }
    
    private int[] computeRange(ArrayReader reader, int target){
        int r = 1;
        while (reader.get(r) < target) {
            r <<= 1;
        }
        
        int l = r >> 1;
        while (r >= l && reader.get(r) == 2147483647) {
            r--;
        }
        if (r < l) {
            return null;
        }
        
        return new int[]{l, r};
    }
    
    private int bsearchLowerBound(ArrayReader reader, int l, int r, int v) {
        while (l < r) {
            int m = l + (r - l) / 2;
            if (reader.get(m) >= v) {
                r = m;
            } else {
                l = m + 1;
            }
        }
        return l;
    }
}
```
