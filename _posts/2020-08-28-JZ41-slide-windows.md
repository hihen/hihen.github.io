---
title: 输出所有和为S的连续正数序列。
author: hihen
date: 2020-08-28 00:58:00 +0800
categories: [算法, 双指针]
tags: 算法
---

```java
public class JZ41 {
    /**
     * solution: slide windows(double point). two points just can move to right.
     *
     * @param sum
     * @return
     */
    public ArrayList<ArrayList<Integer>> FindContinuousSequence(int sum) {
        ArrayList<ArrayList<Integer>> res = new ArrayList<>();
        int lowNum = 1, highNum = 2;
        while (lowNum < highNum) {
            // expression that calculate sum between A1 and An: (A1+An)*n/2
            int curSum = (lowNum + highNum) * (highNum - lowNum + 1) / 2;
            if (curSum == sum) {
                ArrayList<Integer> list = new ArrayList<>();
                for (int i = lowNum; i <= highNum; i++) {
                    list.add(i);
                }
                res.add(list);
                lowNum++;
            } else if (curSum < sum) {
                highNum++;
            } else {
                // curSum > sum, why need lowNum++, why not highNum--, because point just can move to right.
                lowNum++;
            }
        }
        return res;
    }
}
```