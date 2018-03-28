---
title: '[LeetCode] 15. 3Sum'
date: 2018-03-23 10:44:37
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Given an array S of n integers, are there elements a, b, c in S such that a + b + c = 0? Find all unique triplets in the array which gives the sum of zero.
    Note: The solution set must not contain duplicate triplets.

### 标签：
`Array`, `Two Pointers`

### 分析：
比较直观的解法是三个嵌套循环直接暴搜解法，最后去重，计算复杂度为$O(n^3)$，不过，暴力枚举过程中存在着大量可以省略的重复计算工作，存在很大的优化空间，事实上最终的优解复杂度仅有$O(n^2)$而已。

1. 首先，将三数和归纳为二数和的求解，即固定一个数a，然后在剩余的数中求解b和c；需要注意的是，a是一次性的，在对a求解完b、c的组合后，a就不需要再被用到；对所有数依次固定求解完之后，再去除重复解，并排序。复杂度为$O(n^2)$（因为二数和求解是$O(n)$）
2. 可以首先对整个数列进行排序，通过假设`a < b < c`来简化计算；具体为，每次固定完数a后，我们只需要往后搜索b、c的解法（参见代码14行），因为a前面的数已经全部被求解过一遍了。
3. 题目要求最后不能有重复解，不建议在最后进行去重，完全可以在计算过程中进行避免；由于数列已经经过排序，所有相同的数一定分在一块，因此，只需要对连续重复的a求解一次即可（参见代码10行）。
4. 由于假设`a < b < c`的存在，我们知道a必须小于等于0，因此，可以当a大于0时，可以终止求解过程（参见代码8行）。
5. 最后，由于数列是排好序的，因此二数和可以从两端同时进行计算求解。
6. 最后的最后，尽可能多用中间变量来缓存计算结果（参见代码13、17行），不要在循环内做重复计算，这会对算法运行时间产生不小的影响。

### 代码：
``` c++
class Solution {
public:
    // try to find all a, b, c where a + b + c = 0
    vector<vector<int>> threeSum(vector<int>& nums) {
        sort(nums.begin(), nums.end()); // sort the number array to ensure a < b < c.
        vector<vector<int>> res;
        for (int i = 0; i < (int)nums.size(); i++) {
            if (nums[i] > 0) // a cannot be positive since both b and c is larger than a.
                break;
            if (i != 0 && nums[i] == nums[i-1]) // skip all duplicate a's.
                continue;

            int target = 0 - nums[i];
            int j = i+1, k = nums.size() - 1;
            while (true) { // since the nums array is sorted, we could search from both sides.
                if (j >= k) break;
                int bPlusC = nums[j] + nums[k];
                if (bPlusC > target) k--; // b + c > 0 - a, so decrease c.
                else if (bPlusC < target) j++; // b + c < 0 - a, so increase b.
                else {
                    vector<int> newSol{nums[i], nums[j], nums[k]};
                    res.push_back(newSol);
                    if (nums[j] == nums[k]) break; // stop since c cannot be smaller than b
                    do {
                       j++;
                    } while (nums[j] == nums[j-1]); // skip the duplicate b
                    do {
                       k--;
                    } while (nums[k] == nums[k+1]); // skip the duplicate c
                }
            }
        }
        return res;
    }
};
```

