---
title: '[LeetCode] 26. Remove Duplicates from Sorted Array'
date: 2018-03-28 09:49:42
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Given a sorted array, remove the duplicates in-place such that each element appear only once and return the new length.
    Do not allocate extra space for another array, you must do this by modifying the input array in-place with O(1) extra memory.

    Example:
    Given nums = [1,1,2],
    Your function should return length = 2, with the first two elements of nums being 1 and 2 respectively.
    It doesn't matter what you leave beyond the new length.

### 标签：
`Array`, `Two Pointers`

### 分析：
数组是排好序的，因此重复数字一定相邻。

### 代码：

``` c++
class Solution {
public:
    int removeDuplicates(vector<int>& nums) {
        if (nums.size() <= 1) return nums.size();
        int i = 0, j = 1;
        while (j < nums.size()) {
            if (nums[i] == nums[j++]) continue;
            else nums[++i] = nums[j-1];
        }
        return i+1;
    }
};
```
