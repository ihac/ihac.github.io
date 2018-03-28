---
title: '[LeetCode] 21. Merge Two Sorted Lists'
date: 2018-03-25 21:35:05
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Merge two sorted linked lists and return it as a new list. The new list should be made by splicing together the nodes of the first two lists.

### 标签：
`Linked List`

### 分析：
将两个有序列表合并为一个有序列表。为了减少逻辑判断，可以新建一个node作为虚拟头部，省去对原头部的单独处理。

### 代码：
``` c++
class Solution {
public:
    ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
        // virtual head.
        ListNode newhead(INT_MIN);
        ListNode* tail = &newhead;
        while (l1 != nullptr && l2 != nullptr) {
            if (l1->val <= l2->val) {
                tail->next = l1;
                l1 = l1->next;
            }
            else {
                tail->next = l2;
                l2 = l2->next;
            }
            tail = tail->next;
        }
        // append the remaining list.
        tail->next = l1 == nullptr? l2 : l1;
        return newhead.next;
    }
};
```

