---
layout:     post
title:      "面试题 04.03. 特定深度节点链表"
subtitle:   ""
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    false
tags:
    - tags
---



```
给定一棵二叉树，设计一个算法，创建含有某一深度上所有节点的链表（比如，若一棵树的深度为 D，则会创建出 D 个链表）。返回一个包含所有深度的链表的数组。

 

示例：

输入：[1,2,3,4,5,null,7,8]

        1
       /  \ 
      2    3
     / \    \ 
    4   5    7
   /
  8

输出：[[1],[2,3],[4,5,7],[8]]
```



```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
```



BFS 98.54%

```java
class Solution {
    public ListNode[] listOfDepth(TreeNode tree) {
        if (tree == null) return new ListNode[0];

        Deque<TreeNode> queue = new ArrayDeque<>();
        queue.add(tree);

        List<ListNode> nodes = new ArrayList<>();
        ListNode dummy = new ListNode(0);

        while (!queue.isEmpty()) {
            int size = queue.size();
            ListNode curr = dummy;

            for (int i = 0; i < size; i++) {
                TreeNode node = queue.poll();

                curr.next = new ListNode(node.val);
                curr = curr.next;

                if (node.left != null) queue.add(node.left);
                if (node.right != null) queue.add(node.right);
            }

            nodes.add(dummy.next);
            dummy.next = null;
        }

        return nodes.toArray(new ListNode[0]);
    }
}
```

