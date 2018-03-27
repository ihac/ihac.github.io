---
title: '[LeetCode] 16. 3Sum Closest'
date: 2018-03-23 21:05:38
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Given an array S of n integers, find three integers in S such that the sum is closest to a given number, target. Return the sum of the three integers. You may assume that each input would have exactly one solution.
    For example, given array S = {-1 2 1 -4}, and target = 1.
    The sum that is closest to the target is 2. (-1 + 2 + 1 = 2).

### 分析：
和15题类似，同样采取固定a，然后求解b、c的最优解，区别在于，这里只需要返回三数之和，因此实际上题目更简单了。
容易出错的点主要是一些临界数据，比如数列刚好仅有三个数时，如果没有做好初始化工作，答案就会出错。

### 代码：
``` c++
class Solution {
public:
    // find a, b, c whose sum is closest to target and return their sum.
    int threeSumClosest(vector<int>& nums, int target) {
        if ((int)nums.size() < 3) // return immediately.
            return 0;

        sort(nums.begin(), nums.end()); // sort the array to ensure a < b < c.
        int res = nums[0] + nums[1] + nums[2]; // initialize result.

        for (int i = 0; i < (int)nums.size(); i++) {
            if (i != 0 && nums[i] == nums[i-1]) // skip the duplicate a.
                continue;
            int bPlusC = target - nums[i];
            int j = i + 1, k = (int)nums.size() - 1;

            while (j < k) {
                int sum = nums[j] + nums[k];
                int delta = sum - bPlusC;
                if (abs(delta) < abs(target - res)) // whether closer to target than previous sum.
                    res = sum + nums[i];

                if (delta < 0) j++; // b + c < 0 - a, so increase b.
                else if (delta > 0) k--; // b + c > 0 - a, so decrease c.
                else return res; // b + c = 0 - a, best answer.
            }
        }
        return res;
    }
};
```

