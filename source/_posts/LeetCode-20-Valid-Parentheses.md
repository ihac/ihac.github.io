---
title: '[LeetCode] 20. Valid Parentheses'
date: 2018-03-25 21:33:34
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Given a string containing just the characters '(', ')', '{', '}', '[' and ']', determine if the input string is valid.
    The brackets must close in the correct order, "()" and "()[]{}" are all valid but "(]" and "([)]" are not.

### 标签：
`String`, `Stack`

### 分析：
本题比较简单，可能的出错点在于对空栈进行pop操作～比如初始字符可能为右括号。

### 代码：
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

