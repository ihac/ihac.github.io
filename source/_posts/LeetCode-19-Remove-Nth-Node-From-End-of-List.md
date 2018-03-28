---
title: '[LeetCode] 19. Remove Nth Node From End of List'
date: 2018-03-24 21:29:16
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Given a linked list, remove the nth node from the end of list and return its head.
    For example,

    Given linked list: 1->2->3->4->5, and n = 2.
    After removing the second node from the end, the linked list becomes 1->2->3->5.

    Note:
    Given n will always be valid.
    Try to do this in one pass.

### 标签：
`Linked List`, `Two Pointers`

### 分析：
给定的是单列表，且要求解法为one-pass，因此可以使用两个指针同时移动，当前面的指针到达链表尾部时，删除后面指针指向的node即可。

### 代码：

```c++
// Definition for singly-linked list.
struct ListNode {
    int val;
    ListNode *next;
    ListNode(int x) : val(x), next(nullptr) {}
};
class Solution {
public:
    ListNode* removeNthFromEnd(ListNode* head, int n) {
        if (head == nullptr) return nullptr;
        ListNode *a, *b;
        a = b = head;
        int i = 1;
        while (i < n) {
            // this should not happen
            if (b->next == nullptr) break;
            b = b->next;
        }

        ListNode* prev = nullptr;
        while (b->next != nullptr) {
            prev = a;
            b = b->next;
            a = a->next;
        }

        if (prev == nullptr)
            head = head->next; // remove head
        else
            prev->next = a->next;
        return head;
    }
};
```

