---
title: 【刷题】Lowest Common Ancestor
tags:
  - LintCode
  - 算法
  - 面试
  - 原创
reward: true
date: 2017-12-17 23:21:33
---

最低公共祖先 Lowest Common Ancestor 三连击.

<!--more-->

# basic problem

* [lintcode戳我](http://www.lintcode.com/en/problem/lowest-common-ancestor/)（有follow up）
* [leetcode戳我](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-tree/)

## 题目

### Description

Given the root and two nodes in a Binary Tree. Find the lowest common ancestor(LCA) of the two nodes.

The lowest common ancestor is the node with largest depth which is the ancestor of both nodes.

>Notice:
>
>Assume two nodes are exist in tree.

### Example

For the following binary tree:

```
  4
 / \
3   7
   / \
  5   6
```

* LCA(3, 5) = 4
* LCA(5, 6) = 7
* LCA(6, 7) = 7


### Tags 

`LinkedIn` `LintCode Copyright` `Binary Tree` `Facebook`

## 分析

在leetcode上直接递归遍历path会爆栈。**估计面试官会要求O(1)的空间复杂度**吧，这样就不会往遍历path的思路上想了。。。

在O(1)的空间复杂度下，思路就比较奇特了，猴子看题解后还想了一会才能明白。

首先，`Given the root and two nodes in a Binary Tree`，表示给定的**两个节点必然在树内**，这一点非常重要，支撑解法的核心思想：

* 将求LCA转化为求Most Possible LCA

下面配合注释看代码。

>可能是我比较迟钝吧，，，我觉得这题适合做follow up，为啥成basic problem了呢。

## 完整代码

```java
class TreeNode {
  int val;
  TreeNode left;
  TreeNode right;

  TreeNode(int x) {
    val = x;
  }
}

// 两节点在树中
public class Solution {
  public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if (root == null) {
      return null;
    }
    // p and q must exist in tree, so that lca of p and q must exist
    return findMostPossibleLCA(root, p, q);
  }

  private TreeNode findMostPossibleLCA(TreeNode root, TreeNode node1, TreeNode node2) {
    if (root == null) {
      return null;
    }
    if (root == node1 || root == node2) {
      return root;
    }

    TreeNode leftPLCA = findMostPossibleLCA(root.left, node1, node2);
    TreeNode rightPLCA = lowestCommonAncestor(root.right, node1, node2);
    if (leftPLCA == null && rightPLCA == null) {
      // must not exist LCA
      return null;
    }
    if (leftPLCA != null && rightPLCA != null) {
      // must be LCA
      return root;
    }
    // most possible be LCA
    return leftPLCA != null ? leftPLCA : rightPLCA;
  }
}
```

# follow up 1

* [lintcode戳我](http://www.lintcode.com/en/problem/lowest-common-ancestor-ii/)

## 题目

### Description

Given the root and two nodes in a Binary Tree. Find the lowest common ancestor(LCA) of the two nodes.

The lowest common ancestor is the node with largest depth which is the ancestor of both nodes.

The node has an extra attribute **`parent`** which point to the father of itself. The root's parent is null.

>Notice:
>
>Assume two nodes are exist in tree.

### Example

For the following binary tree:

```
  4
 / \
3   7
   / \
  5   6
```

* LCA(3, 5) = 4
* LCA(5, 6) = 7
* LCA(6, 7) = 7


### Tags 

`LintCode Copyright` `Binary Tree`

## 分析

两节点仍然在树中，**增加了parent指针**。

有parent指针后，时间复杂度能降低到O(lgn)，空间复杂度O(1)。

>利用了链表题中的**长度差**同步技巧。

## 完整代码

```java
class ParentTreeNode {
  public ParentTreeNode parent, left, right;
}

public class FollowUp1 {
  // 1. 先分别向上遍历到root，得到两个深度d1，d2
  // 2. 回到节点位置，更深的先向上走abs(d1-d2)步
  // 3. 然后二者一起走min(d1,d2)步，过程中一定会有根节点
  // 时间O(lgn)，空间O(1)
  public ParentTreeNode lowestCommonAncestor(ParentTreeNode root,
                                             ParentTreeNode p,
                                             ParentTreeNode q) {
    ParentTreeNode node1 = p;
    ParentTreeNode node2 = q;
    if (root == null || node1 == null || node2 == null) {
      return null;
    }

    int depth1 = getDepth(root, node1);
    int depth2 = getDepth(root, node2);
    if (depth1 == -1 || depth2 == -1) {
      return null;
    }

    ParentTreeNode startNode1 = node1;
    ParentTreeNode startNode2 = node2;
    int depth = depth1;
    if (depth1 > depth2) {
      for (int i = 0; i < depth1 - depth2; i++) {
        startNode1 = startNode1.parent;
      }
      depth = depth2;
    } else if (depth1 < depth2) {
      for (int i = 0; i < depth2 - depth1; i++) {
        startNode2 = startNode2.parent;
      }
      depth = depth1;
    }

    for (int i = 0; i < depth; i++) {
      if (startNode1 == startNode2) {
        return startNode1;
      }
      startNode1 = startNode1.parent;
      startNode2 = startNode2.parent;
    }

    throw new RuntimeException("UnknownError");
  }

  private int getDepth(ParentTreeNode root, ParentTreeNode target) {
    int depth = 1;
    ParentTreeNode node = target;
    for (; node.parent != null; node = node.parent) {
      if (node == root) {
        break;
      }
      depth++;
    }

    if (node == root) {
      return depth;
    }
    return -1;
  }
}
```

# follow up 2

* [lintcode戳我](http://www.lintcode.com/en/problem/lowest-common-ancestor-iii/)

## 题目

### Description

Given the root and two nodes in a Binary Tree. Find the lowest common ancestor(LCA) of the two nodes.

The lowest common ancestor is the node with largest depth which is the ancestor of both nodes.

Return null if LCA does not exist.

>Notice:
>
>node A or node B may not exist in tree.

### Example

For the following binary tree:

```
  4
 / \
3   7
   / \
  5   6
```

* LCA(3, 5) = 4
* LCA(5, 6) = 7
* LCA(6, 7) = 7


### Tags 

`LinkedIn` `LintCode Copyright` `Binary Tree` `Facebook`

## 分析

两节点可能不在树中，节点也没有parent指针。

由于没有parent指针，那么根据树遍历找到节点至少是O(n)的时间复杂度。同时，还要花费O(n)的空间复杂度记录路径。

## 完整代码

```java
public class FollowUp2 {
  public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    TreeNode node1 = p;
    TreeNode node2 = q;
    if (root == null || node1 == null || node2 == null) {
      return null;
    }

    Stack<TreeNode> path1 = new Stack<>();
    if (!dfsPreorder(root, node1, path1)) {
      return null;
    }
    Stack<TreeNode> path2 = new Stack<>();
    if (!dfsPreorder(root, node2, path2)) {
      return null;
    }

    TreeNode lca = null;
    for (int i = 0; i < path1.size() && i < path2.size(); i++) {
      if (path1.get(i) != path2.get(i)) {
        break;
      }
      lca = path1.get(i);
    }

    return lca;
  }

  private boolean dfsPreorder(TreeNode root, TreeNode node, Stack<TreeNode> path) {
    path.push(root);
    if (root == node) {
      return true;
    }
    if (root.left != null && dfsPreorder(root.left, node, path)) {
      return true;
    }
    if (root.right != null && dfsPreorder(root.right, node, path)) {
      return true;
    }
    path.pop();
    return false;
  }
}
```
