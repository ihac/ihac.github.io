---
title: '[LeetCode] 32. Longest Valid Parentheses'
date: 2018-04-20 09:35:17
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目

    Given a string containing just the characters '(' and ')', find the length of the longest valid (well-formed) parentheses substring.


### 标签
`String`, `Dynamic Programming`, `Stack`

### 分析

TBD

### 代码

``` c++
class Solution {
public:
    /*
     * two traverse
     */
    int longestValidParentheses(string s) {
        int left, right, stack, len;
        left = len = 0;
        // traverse from left to right
        while (left < (int)s.length()) {
            while (left < (int)s.length() && s[left] == ')') left++;
            right = left;
            stack = 0;
            while (right < (int)s.length()) {
                // stack = Nums(left parentheses) - Nums(right parentheses)
                if (s[right] == '(') stack++;
                else if (s[right] == ')') {
                    stack--;
                    if (stack == 0) len = max(len, right - left - stack + 1);
                    else if (stack < 0) break;
                }
                right++;
            }
            left = right;
        }

        right = (int)s.length() - 1;
        // traverse from right to left
        while (right >= 0) {
            while (right >= 0 && s[right] == '(') right--;
            left = right;
            stack = 0;
            while (left >= 0) {
                // stack = Nums(right parenth) - Nums(left parenth)
                if (s[left] == ')') stack++;
                else if (s[left] == '(') {
                    stack--;
                    if (stack == 0) len = max(len, right - left - stack + 1);
                    else if (stack < 0) break;
                }
                left--;
            }
            right = left;
        }
        return len;
    }

    /*
     * using stack
     */
    int longestValidParentheses_stack(string s) {
        stack<int> st;
        int len = 0;
        // always store the start index in the bottom of stack.
        // -1 at start, index of last unmatched right parenthesis when traversing.
        st.push(-1);
        for (int i = 0; i < (int)s.length(); i++) {
            if (s[i] == '(') st.push(i);
            else {
                st.pop();
                // empty means all left parentheses were run out
                if (st.empty()) st.push(i);
                else len = max(len, i - st.top());
            }
        }
        return len;
    }


    /*
     * using dynamic programming
     */
    int longestValidParentheses_dp(string s) {
        if (s.length() <= 1) return 0;
        int dpArray[s.length() + 1];
        dpArray[0] = dpArray[1] = 0;
        dpArray[2] = (s[0] == '(' && s[1] == ')')? 2 : 0;
        int len = dpArray[2];
        for (int i = 2; i < (int)s.length(); i++) {
            int ii = i + 1;
            if (s[i] == '(') dpArray[ii] = 0;
            else {
                if (s[i-1] == '(') dpArray[ii] = dpArray[ii-2] + 2;
                else {
                    if (i > dpArray[ii-1] && s[i - dpArray[ii-1] - 1] == '(')
                        dpArray[ii] = dpArray[ii-1] + 2 + dpArray[i-dpArray[ii-1]-1];
                    else dpArray[ii] = 0;
                }
            }
            len = max(len, dpArray[ii]);
        }
        return len;
    }
};k
```
