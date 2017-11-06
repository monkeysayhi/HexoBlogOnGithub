---
title: 【刷题】strStr II - Rabin Karp
date: 2017-08-06 00:54:20
tags: 
  - LintCode
  - 算法
  - 面试
  - 原创
reward: true
---

[原题戳我](http://www.lintcode.com/en/problem/strstr-ii)

<!--more-->

介绍另一种更通用的算法，可以代替KMP以O(n+m)的时间复杂度完成字符串查找问题。

# KMP

本科一般都学习过KMP算法，它能在O(n+m)的时间内解决字符串查找问题，不赘述，可参考[KMP戳我](https://zh.wikipedia.org/wiki/%E5%85%8B%E5%8A%AA%E6%96%AF-%E8%8E%AB%E9%87%8C%E6%96%AF-%E6%99%AE%E6%8B%89%E7%89%B9%E7%AE%97%E6%B3%95)。

很容易理解，KMP已经是效率最高的字符串查找算法。整个算法的重点在next数组的生成上，该过程不是很难理解，实现起来却不太方便，又没什么通用性，特意去记忆的性价比太低。**不管在面试还是实际问题中，都不是一个很好的选择。**

> Java String类中的indexOf()方法，C++的strstr()函数，使用了O(n*m)的暴力比较；Golang strings包中的Index()方法中使用了下述Rabin-Karp算法，时间复杂度O(n+m)，同样没有使用KMP。

# Rabin-Karp

为了保证O(n+m)的时间复杂度，可以使用更通用的Rabin-Karp算法。

算法非常简单，总结为三点：

1. 用hash的比较代替字符串的比较，时间复杂度为O(1)（一般假设hash不碰撞）
2. 需要提前计算出初始的srcHash和固定的tgtHash，时间复杂度为O(m)
3. 待比较的新srcStr是渐变的，计算新srcHash的时间复杂度为O(1)

因此，Rabin-Karp的时间复杂度也是O(n+m)。详细可参考[Rabin-Karp戳我](http://www.geeksforgeeks.org/searching-for-patterns-set-3-rabin-karp-algorithm/)。

也就没什么可说的了，上代码：

```java
public int strStr2(String source, String target) {
	if (source == null || target == null) {
		return -1;
	}
	if (source.length() < target.length()) {
		return -1;
	}
	if (target.length() == 0) {
		return 0;
	}

	final int MAGIC_NUM = 31;
	final int MODE = 1000007;

	int highestPower = 1;
	for (int i = 0; i < target.length(); i++) {
		highestPower = (highestPower * MAGIC_NUM) % MODE;
	}

	// init sourceHash and targetHash
	int sourceHash = 0;
	int targetHash = 0;
	for (int i = 0; i < target.length(); i++) {
		sourceHash = (((sourceHash + source.charAt(i)) % MODE) * MAGIC_NUM) % MODE;
		targetHash = (((targetHash + target.charAt(i)) % MODE) * MAGIC_NUM) % MODE;
	}

	// "i + (target.length() - 1) < source.length()" is for limit of "i + j"
	for (int i = 0; i + (target.length() - 1) < source.length(); i++) {
		//update sourceHash
		if (i - 1 >= 0) {
			// for this problem, pre-calculating highestPower is necessary to avoid TLE...T_T
			int minus = (source.charAt(i - 1) * highestPower) % MODE;
			sourceHash = (sourceHash + (MODE - minus)) % MODE;
			sourceHash = (((sourceHash + source.charAt(i + target.length() - 1))  % MODE) * MAGIC_NUM) % MODE;
		}
		//judge
		if (sourceHash == targetHash) {
			for (int j = 0; j < target.length(); j++) {
				if (source.charAt(i + j) != target.charAt(j)) {
					return -1;
				}
			}
			return i;
		}
	}

	return -1;
}
```

需要注意的是：

* hash还是可能碰撞，因此，当hash相等时，还是需要扫描一遍确认是否真的相等
* 我们前面直接认为“hash的计算是O(1)的”，实际上，不是任意一个hash函数都能在常数时间内随着字符串的渐变而更新，别随便选hash函数

# 引申

Rabin-Karp算法专注于O(1)的比较，可以扩展到其他几乎所有渐变状态的比较上面。比如八数码问题，相邻状态只有两个位置不同，是渐变的，便可以使用Rabin-Karp算法将比较时间降到O(1)。同时，算法又极其简单，理解起来完全无障碍，随手就能实现。除了特定情况，我个人建议多使用基于Robin-Karp算法的字符串查找。
