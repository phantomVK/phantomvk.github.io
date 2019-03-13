---
layout:     post
title:      "LeetCode-53 最大子序和"
date:       2019-04-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Algorithm
---

## 问题

给定一个整数数组 `nums` ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

**示例:**

```
输入: [-2,1,-3,4,-1,2,1,-5,4],
输出: 6
解释: 连续子数组 [4,-1,2,1] 的和最大，为 6。
```

**进阶:**

如果你已经实现复杂度为 O(*n*) 的解法，尝试使用更为精妙的分治法求解。

## 解答

```java
public int maxSubArray(int[] nums) {
    int dp = nums[0];
    int max = dp;
    for (int i = 1; i < nums.length; ++i) {
        dp = nums[i] + (dp > 0 ? dp : 0);
        max = Math.max(max, dp);
    }

    return max;
}
```



