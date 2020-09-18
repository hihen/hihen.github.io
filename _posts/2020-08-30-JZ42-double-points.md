---
title: 输入一个递增排序的数组和一个数字S，在数组中查找两个数，使得他们的和正好是S，如果有多对数字的和等于S，输出两个数的乘积最小的。
author: hihen
date: 2020-08-30 12:30:00 +0800
categories: [算法, 双指针]
tags: 算法
---

```java
public class JZ42 {
    /**
     * solution: double points, one is in the array leftmost, other is in the array rightmost.
     * two points move to centre gradually.
     *
     * @param array
     * @param sum
     * @return
     */
    public ArrayList<Integer> FindNumbersWithSum(int[] array, int sum) {
        ArrayList<Integer> list = new ArrayList<>();
        int lowNum = 0, highNum = array.length - 1;
        while (lowNum < highNum) {
            int tempSum = array[lowNum] + array[highNum];
            // because numbers in the side multi result less more numbers in the centre
            // so, if get equal, just direct return, them must be min multi.
            if (tempSum == sum) {
                list.add(array[lowNum]);
                list.add(array[highNum]);
                return list;
            } else if (tempSum < sum) {
                lowNum++;
            } else {
                highNum--;
            }
        }
        return list;
    }
}
```