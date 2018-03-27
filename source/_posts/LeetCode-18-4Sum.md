---
title: '[LeetCode] 18. 4Sum'
date: 2018-03-24 21:20:22
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：
    Given an array S of n integers, are there elements a, b, c, and d in S such that a + b + c + d = target? Find all unique quadruplets in the array which gives the sum of target.
    Note: The solution set must not contain duplicate quadruplets.

### 分析：
四数和解法和三数和没有本质区别，无非是多固定一个b而已，复杂度为$O(n^3)$。不过为了优化算法性能，我们需要做一些剪枝，比如忽略重复的a（代码17行）、跳过过小的a（代码19行）以及a过大时结束循环（代码21行）。

### 代码：
``` c++
class Solution {
public:
    // find all unique a, b, c, d where a + b + c + d = target.
    vector<vector<int>> fourSum(vector<int>& nums, int target) {
        vector<vector<int>> res;
        if (nums.size() < 4) return res;

        // do not modify the input vector.
        vector<int> nums_copy(nums.begin(), nums.end());
        // sort the nums array to ensure a <= b <= c <= d;
        sort(nums_copy.begin(), nums_copy.end());

        int size = nums_copy.size();
        int last3sum = nums_copy[size-1] + nums_copy[size-2] + nums_copy[size-3];
        int last2sum = nums_copy[size-1] + nums_copy[size-2];
        for (int i = 0; i < size-3; i++) {
            // skip the duplicate a
            if (i != 0 && nums_copy[i] == nums_copy[i-1]) continue;
            // a is too small.
            if (nums_copy[i] + last3sum < target) continue;
            // a is too large.
            if (nums_copy[i] + nums_copy[i+1] + nums_copy[i+2] + nums_copy[i+3] > target) break;

            int target1 = target - nums_copy[i];
            for (int j = i+1; j < size-2; j++) {
                // skip the duplicate b.
                if (j != i+1 && nums_copy[j] == nums_copy[j-1]) continue;
                // b is too small.
                if (nums_copy[j] + last2sum < target1) continue;
                // b is too large.
                if (nums_copy[j] + nums_copy[j+1] + nums_copy[j+2] > target1) break;

                int target2 = target1 - nums_copy[j];
                int k = j + 1, l = size - 1;
                while (k < l) {
                    int sum = nums_copy[k] + nums_copy[l];
                    if (sum < target2) k++;
                    else if (sum > target2) l--;
                    else {
                        vector<int> tmp{nums_copy[i], nums_copy[j], nums_copy[k], nums_copy[l]};
                        res.push_back(tmp);
                        do { k++; } while (nums_copy[k] == nums_copy[k-1]);
                        do { l--; } while (nums_copy[l] == nums_copy[l+1]);
                    }
                }
            }
        }
        return res;
    }
};
```

