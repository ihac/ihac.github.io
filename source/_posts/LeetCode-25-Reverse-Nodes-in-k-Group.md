---
title: '[LeetCode] 25. Reverse Nodes in k-Group'
date: 2018-03-28 09:39:46
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Given a linked list, reverse the nodes of a linked list k at a time and return its modified list.
    k is a positive integer and is less than or equal to the length of the linked list. If the number of nodes is not a multiple of k then left-out nodes in the end should remain as it is.
    You may not alter the values in the nodes, only nodes itself may be changed.
    Only constant memory is allowed.

    For example,
    Given this linked list: 1->2->3->4->5
    For k = 2, you should return: 2->1->4->3->5
    For k = 3, you should return: 3->2->1->4->5

### 标签：
`Linked List`

### 分析：
可以看作$\frac{n}{k}$个长度为$k$的链表分别进行反转，总的计算复杂度为$O(n)$。题目要求不反转最后剩余的、个数少于$k$的节点，因此可以预先遍历整个链表，获取总长度；但更好的一种方法是，不管剩余节点个数，每过$k$个节点就进行一轮新的反转，不过对于最后剩余的节点进行两轮反转，恢复原序。

### 代码：
``` c++
class Solution {
public:
    ListNode* reverseKGroup(ListNode* head, int k) {
        ListNode vhead(INT_MIN);
        vhead.next = head;
        ListNode *newhead = &vhead, *curr = &vhead, *tail, *nxt;
        int remLen = 0;
        while (curr = curr->next) // find the length of the list.
            remLen++;

        while (remLen >= k) {
            curr = newhead->next;
            for (int i = 1; i < k; i++) {
                nxt = curr->next;
                curr->next = nxt->next;
                nxt->next = newhead->next;
                newhead->next = nxt;
            }
            newhead = curr;
            remLen -= k; // subtract k after each iteration.
        }
        return vhead.next;
    }
};
```
