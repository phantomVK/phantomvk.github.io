---
layout:     post
title:      "链表逆序"
date:       2019-01-01
author:     "phantomVK"
header-img: "img/bg/post_bg.jpg"
catalog:    true
tags:
    - Algorithm
---

```java
public class ReverseLinkedList {

    public static Node reverseLinkedList(Node node) {
        Node prev = null;
        Node curr;
        Node next;

        // 链表为空或链表只有一个节点，则无需逆序
        if (node.getNext() == null || node.getNext().getNext() == null) {
            return node;
        }

        curr = node.getNext();
        while (curr != null) {
            next = curr.getNext();
            curr.setNext(prev);
            prev = curr;
            curr = next;
        }

        node.setNext(prev);
        return node;

    }
}

class Node {
    private int mData;
    private Node mNext;

    public int getData() {
        return mData;
    }

    public void setData(int data) {
        mData = data;
    }

    public Node getNext() {
        return mNext;
    }

    public void setNext(Node next) {
        mNext = next;
    }
}
```

