---
title: 判断二叉树是否为平衡二叉树
author: hihen
date: 2020-08-19 20:55:00 +0800
categories: [算法, 树]
tags: 算法
---

```java
 public class JZ39 {
        // 平衡二叉树定义：对于树中任意一个节点，它的两个子树的高度差<=1
        // 解法：通过深度优先搜索算法，递归的求出两个子节点的高度，比较高度差是否超过1
        public boolean IsBalanced_Solution(TreeNode root) {
            int high = calcHigh(root);
            return high != -1;
        }
    
        private int calcHigh(TreeNode node) {
            if (node == null) {
                return 0;
            }
            int leftHigh = calcHigh(node.left);
            // 当其中某个子节点的遍历返回-1，代表程序不满足平衡二叉树，直接得出结论，不继续遍历后面的节点了
            if (leftHigh == -1) {
                return -1;
            }
            int rightHigh = calcHigh(node.right);
            if (rightHigh == -1) {
                return -1;
            }
            // 如果高度差超过1，就返回-1，代表失败
            return Math.abs(leftHigh - rightHigh) > 1 ? -1 : Math.max(leftHigh, rightHigh) + 1;
        }
 }
```

