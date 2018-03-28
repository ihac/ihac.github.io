---
title: '[LeetCode] 24. Swap Nodes in Pairs'
date: 2018-03-26 21:39:14
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Given a linked list, swap every two adjacent nodes and return its head.
    For example,
    Given 1->2->3->4, you should return the list as 2->1->4->3.

    Your algorithm should use only constant space. You may not modify the values in the list, only nodes itself can be changed.

### 标签：
`Linked List`

### 分析：
尝试不用虚头部解决这类链表问题～在正确性得到保证的情况下，尽可能地保持代码简洁。

### 代码：
``` c++
class Solution {
public:
    ListNode* swapPairs(ListNode* head) {
        ListNode **hhead = &head;
        ListNode *a, *b;
        while ((a = *hhead) && (b = a->next)) {
            a->next = b->next;
            b->next = a;
            *hhead = b;

            hhead = &(a->next);
        }
        return head;
    }
};
```

