---
title: LeetCode解题报告
tags: [Algorithm, LeetCode, C/C++]
---

<!-- date: 2018-03-23 14:29 -->
### 15. 3Sum
题目：

    Given an array S of n integers, are there elements a, b, c in S such that a + b + c = 0? Find all unique triplets in the array which gives the sum of zero.
    Note: The solution set must not contain duplicate triplets.

分析：
比较直观的解法是三个嵌套循环直接暴搜解法，最后去重，计算复杂度为$O(n^3)$，不过，暴力枚举过程中存在着大量可以省略的重复计算工作，存在很大的优化空间，事实上最终的优解复杂度仅有$O(n^2)$而已。

1. 首先，将三数和归纳为二数和的求解，即固定一个数a，然后在剩余的数中求解b和c；需要注意的是，a是一次性的，在对a求解完b、c的组合后，a就不需要再被用到；对所有数依次固定求解完之后，再去除重复解，并排序。复杂度为$O(n^2)$（因为二数和求解是$O(n)$）
2. 可以首先对整个数列进行排序，通过假设`a < b < c`来简化计算；具体为，每次固定完数a后，我们只需要往后搜索b、c的解法（参见代码14行），因为a前面的数已经全部被求解过一遍了。
3. 题目要求最后不能有重复解，不建议在最后进行去重，完全可以在计算过程中进行避免；由于数列已经经过排序，所有相同的数一定分在一块，因此，只需要对连续重复的a求解一次即可（参见代码10行）。
4. 由于假设`a < b < c`的存在，我们知道a必须小于等于0，因此，可以当a大于0时，可以终止求解过程（参见代码8行）。
5. 最后，由于数列是排好序的，因此二数和可以从两端同时进行计算求解。
6. 最后的最后，尽可能多用中间变量来缓存计算结果（参见代码13、17行），不要在循环内做重复计算，这会对算法运行时间产生不小的影响。

代码：
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

### 16. 3Sum Closest

题目：

    Given an array S of n integers, find three integers in S such that the sum is closest to a given number, target. Return the sum of the three integers. You may assume that each input would have exactly one solution.
    For example, given array S = {-1 2 1 -4}, and target = 1.
    The sum that is closest to the target is 2. (-1 + 2 + 1 = 2).

分析：
和15题类似，同样采取固定a，然后求解b、c的最优解，区别在于，这里只需要返回三数之和，因此实际上题目更简单了。
容易出错的点主要是一些临界数据，比如数列刚好仅有三个数时，如果没有做好初始化工作，答案就会出错。

代码：
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

### 17. Letter Combinations of a Phone Number

题目：

    Given a digit string, return all possible letter combinations that the number could represent.
    A mapping of digit to letters (just like on the telephone buttons) is given below.
![](http://upload.wikimedia.org/wikipedia/commons/thumb/7/73/Telephone-keypad2.svg/200px-Telephone-keypad2.svg.png)

分析：
本题有多种不同解法:
1. 为了求解长度为n的电话号码所对应的字母组合，我们可以先计算出后n-1位的所有字母组合，然后再与第1位数字对应的多个字母进行组合，如此递归即可求得最终解。这种解法思路比较直观，需要产生n次函数调用，但每次递归结束需要遍历中间结果（vector），然后与当前数字的多个数字进行组合。
2. 采用回溯法，基于深度优先对所有数字进行遍历。产生约$3^n$次函数调用，但是减少了vector的构造、回收与遍历。

代码：
``` c++
// backtracking
class Solution {
public:
    char lookupTable[10][4] = {
        {}, {}, {'a','b','c'}, {'d','e','f'}, {'g','h','i'}, {'j','k','l'}, {'m','n','o'}, {'p','q','r','s'}, {'t','u','v'}, {'w','x','y','z'}
    };
    int length[10] = {
        0, 0, 3, 3, 3, 3, 3, 4, 3, 4
    };
    vector<string> letterCombinations(string digits) {
        if (digits.length() == 0) return vector<string>{};
        vector<string> res;
        string init;
        backtrack(init, res, digits, 0);
        return res;
    }

    void backtrack(string& prev, vector<string>& res, string& digits, int index) {
        if (index == (int)digits.length()) res.push_back(prev);
        else {
            int num = digits[index] - '0';
            for (int i = 0; i < length[num]; i++) {
                prev.push_back(lookupTable[num][i]);
                backtrack(prev, res, digits, index+1);
                prev.pop_back();
            }
        }
    }
};
```

### 18. 4Sum

题目：

    Given an array S of n integers, are there elements a, b, c, and d in S such that a + b + c + d = target? Find all unique quadruplets in the array which gives the sum of target.
    Note: The solution set must not contain duplicate quadruplets.

分析：
四数和解法和三数和没有本质区别，无非是多固定一个b而已，复杂度为$O(n^3)$。不过为了优化算法性能，我们需要做一些剪枝，比如忽略重复的a（代码17行）、跳过过小的a（代码19行）以及a过大时结束循环（代码21行）。

代码：
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

### 19. Remove Nth Node From End of List

题目：

    Given a linked list, remove the nth node from the end of list and return its head.
    For example,

    Given linked list: 1->2->3->4->5, and n = 2.
    After removing the second node from the end, the linked list becomes 1->2->3->5.

    Note:
    Given n will always be valid.
    Try to do this in one pass.

分析：
给定的是单列表，且要求解法为one-pass，因此可以使用两个指针同时移动，当前面的指针到达链表尾部时，删除后面指针指向的node即可。

代码：

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

### 20. Valid Parentheses

题目：

    Given a string containing just the characters '(', ')', '{', '}', '[' and ']', determine if the input string is valid.
    The brackets must close in the correct order, "()" and "()[]{}" are all valid but "(]" and "([)]" are not.

分析：
本题比较简单，可能的出错点在于对空栈进行pop操作～比如初始字符可能为右括号。

代码：

```c++
class Solution {
public:
    bool isValid(string s) {
        map<char, int> lookup;
        lookup['('] = 0;
        lookup[')'] = 1;
        lookup['{'] = 2;
        lookup['}'] = 3;
        lookup['['] = 4;
        lookup[']'] = 5;

        stack<int> par;
        for (int i = 0; i < s.length(); i++) {
            if (lookup[s[i]] % 2 == 0) // for left parentheses, push.
                par.push(lookup[s[i]]);
            else if (par.empty() || par.top() != lookup[s[i]] - 1) // for right parentheses, check and pop.
                return false;
            else
                par.pop();
        }
        return par.empty();
    }
};
```

### 21. Merge Two Sorted Lists

题目：

    Merge two sorted linked lists and return it as a new list. The new list should be made by splicing together the nodes of the first two lists.

分析：
将两个有序列表合并为一个有序列表。为了减少逻辑判断，可以新建一个node作为虚拟头部，省去对原头部的单独处理。

代码：
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
