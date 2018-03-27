---
title: '[LeetCode] 23. Merge k Sorted Lists'
date: 2018-03-26 21:37:52
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Merge k sorted linked lists and return it as one sorted list. Analyze and describe its complexity.

### 分析：
算法流程和Merge 2 Sorted Lists在本质上没有太大区别，都是在多条列表头部中取最小，然后进入下一轮比较，真正影响算法效率的是比较算法。
假设每条列表长度为$n$，列表个数为$k$：
1. 直接遍历比较：那么比较一次需要$k$次操作，需要k次比较才能进入下一轮（即所有列表长度变为$n-1$），总共$n$轮，因此计算复杂度为$k \times k \times n = O(nk^2)$；
2. 采用优先队列（最小堆）：每次取最小值只需要常数次操作，每次新插节点需要$log(k)$次操作，经过$k$次比较进入下一轮，总共$n$轮，因此计算复杂度为$O(k \times log(k) \times n = O(nklog(k)))$；

还有一种思路完全不同的算法：将所有列表两两合并，依次迭代，得到最终解。同样，第一轮中，合并任意两个列表需要$2n$次操作，总共$k/2$次合并；第二轮中，合并两个列表需要$4n$次操作，总共$k/4$次操作；依次类推，总共$log(k)$轮，因此计算复杂度为$O(nklog(k))$。

### 代码：
``` c++
struct cmp{
  bool operator()(ListNode* a, ListNode* b){
    return a->val > b->val;
  }
};
class Solution {
public:
    ListNode* mergeKLists(vector<ListNode*>& lists) {
        ListNode newhead(INT_MIN);
        ListNode *tail = &newhead;

        // min heap.
        priority_queue<ListNode*, vector<ListNode*>, cmp> pq;
        for (int i = 0; i < (int)lists.size(); i++) {
            if (lists[i] != nullptr)
                pq.push(lists[i]);
        }

        while (!pq.empty()) { // stop when all nodes run out.
            ListNode *tmp = pq.top();
            tail->next = tmp;
            tail = tail->next;
            pq.pop();
            if (tmp->next != nullptr)
                pq.push(tmp->next);
        }
        return newhead.next;
    }
};
```
