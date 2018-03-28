---
title: '[LeetCode] 29. Divide Two Integers'
date: 2018-03-28 14:58:39
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Divide two integers without using multiplication, division and mod operator.
    If it is overflow, return MAX_INT.

### 标签：
`Math`, `Binary Search`, `Overflow`

### 分析：
题目看起来很简单，但是通过率并不高。首先，单纯地用dividend一直减divisor肯定会超时，其次，实际计算过程中很容易出现overflow的情况，比如，对INT_MIN求绝对值。为了防止算法超时，可以使用移位运算符成倍增大divisor，尽可能逼近dividend，然后将dividend设为`dividend - divisor`，重复循环。原理与二分查找类似。

### 代码：

``` c++
class Solution {
public:
    int divide(int dividend, int divisor) {
        if (divisor == 0) return INT_MAX;
        if (divisor == 1) return dividend;
        if (divisor == -1) return dividend == INT_MIN? INT_MAX : -dividend;

        // convert both dividend and divisor to negative number.
        bool neg = (dividend < 0) ^ (divisor < 0);
        if (dividend > 0) dividend = -dividend;
        if (divisor > 0) divisor = -divisor;

        int res = 0;
        while (dividend <= divisor) {
            long long cnt = 1, new_divisor = divisor; // use long long to avoid overflow
            while (dividend <= new_divisor) {
                cnt <<= 1;          // cnt *= 2
                new_divisor <<= 1;  // new_divisor *= 2
            }
            dividend -= new_divisor >> 1;
            res += cnt >> 1;
        }
        return neg? -res : res;
    }
};
```
