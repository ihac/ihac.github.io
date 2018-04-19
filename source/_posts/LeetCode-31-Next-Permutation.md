---
title: '[LeetCode] 31. Next Permutation'
date: 2018-04-19 10:06:10
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目

    Implement next permutation, which rearranges numbers into the lexicographically next greater permutation of numbers.
    If such arrangement is not possible, it must rearrange it as the lowest possible order (ie, sorted in ascending order).
    The replacement must be in-place and use only constant extra memory.
    Here are some examples. Inputs are in the left-hand column and its corresponding outputs are in the right-hand column.

    1,2,3 → 1,3,2
    3,2,1 → 1,2,3
    1,1,5 → 1,5,1



### 标签
`Array`

### 分析
先定义在本题中，逆序对表示相邻，且前者比后者更小的元素对。题目解法为：从后往前遍历数组，寻找第一个逆序对（nums[i] < nums[i+1])，此时可知i以后的所有元素按从大到小的顺序排列，我们只需将它们翻转过来，然后将其中最小但比nums[i]大的元素与nums[i]互换即可。当然，如果找不到任何逆序对，那么直接翻转整个数组。


### 代码

``` c++
class Solution {
public:
    void nextPermutation(vector<int>& nums) {
        int i;
        for (i = nums.size() - 2; i >= 0; i--) {
            // stop when we find an inversion pair.
            if (nums[i] < nums[i+1]) {
                // nums[i+1:] is in reverse order.
                reverse(nums.begin() + i + 1, nums.end());
                int tmp = nums[i];
                for (int j = i + 1; j < (int)nums.size(); j++) {
                    // swap nums[i] with the smallest number that is larger than nums[i].
                    if (nums[j] > nums[i]) {
                        nums[i] = nums[j];
                        nums[j] = tmp;
                        break;
                    }
                }
                break;
            }
        }
        // no inversion pair found.
        if (i < 0) reverse(nums.begin(), nums.end());
    }
};
```
