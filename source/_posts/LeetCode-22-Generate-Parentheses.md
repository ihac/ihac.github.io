---
title: '[LeetCode] 22. Generate Parentheses'
date: 2018-03-26 21:36:29
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Given n pairs of parentheses, write a function to generate all combinations of well-formed parentheses.
    For example, given n = 3, a solution set is:

    [
      "((()))",
      "(()())",
      "(())()",
      "()(())",
      "()()()"
    ]

### 分析：
采取回溯法，基于深度优先策略遍历所有括号排列的可能性，唯一的限制在于，当栈为空时，只能插入左括号。

### 代码：
```c++
class Solution {
public:
    vector<string> generateParenthesis(int n) {
        vector<string> res;
        backtrack(res, "", n, n, 0);
        return res;
    }
    void backtrack(vector<string> &res, const string &prev, int l_left, int r_left, int total) {
        if (l_left == 0) { // left parentheses run out.
            res.push_back(prev + string(r_left, ')'));
            return;
        }
        backtrack(res, prev+"(", l_left-1, r_left, total+1);
        if (total > 0) // only when there are left parentheses in stack, can we input a right parenthesis.
            backtrack(res, prev+")", l_left, r_left-1, total-1);
    }
};
```

