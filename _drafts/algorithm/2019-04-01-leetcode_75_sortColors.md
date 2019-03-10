---
layout:     post
title:      "LeetCode-75 颜色分类"
date:       2019-04-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Algorithm
---

## 问题

给定一个包含红色、白色和蓝色，一共 *n* 个元素的数组，**原地**对它们进行排序，使得相同颜色的元素相邻，并按照红色、白色、蓝色顺序排列。

此题中，我们使用整数 0、 1 和 2 分别表示红色、白色和蓝色。

**注意:**
不能使用代码库中的排序函数来解决这道题。

**示例:**

```
输入: [2,0,2,1,1,0]
输出: [0,0,1,1,2,2]
```

**进阶：**

- 一个直观的解决方案是使用计数排序的两趟扫描算法。
  首先，迭代计算出0、1 和 2 元素的个数，然后按照0、1、2的排序，重写当前数组。
- 你能想出一个仅使用常数空间的一趟扫描算法吗？

## 答案

```java
public static void sortColors(int[] arr) {
    if (arr == null || arr.length == 0) return;

    int left = 0;
    int right = arr.length - 1;
    int curr = 0;

    while (curr <= right) {
        if (arr[curr] == 0) {
            swap(arr, left, curr);
            curr++;
            left++;
        } else if (arr[curr] == 2) {
            swap(arr, right, curr);
            right--;
        } else {
            curr++;
        }
    }
}

private static void swap(int[] arr, int a, int b) {
    int temp = arr[a];
    arr[a] = arr[b];
    arr[b] = temp;
}
```

