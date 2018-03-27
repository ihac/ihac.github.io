---
title: '[LeetCode] 17. Letter Combinations of a Phone Number'
date: 2018-03-27 21:12:41
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：
    Given a digit string, return all possible letter combinations that the number could represent.
    A mapping of digit to letters (just like on the telephone buttons) is given below.
![](http://upload.wikimedia.org/wikipedia/commons/thumb/7/73/Telephone-keypad2.svg/200px-Telephone-keypad2.svg.png)

### 分析：
本题有多种不同解法:
1. 为了求解长度为n的电话号码所对应的字母组合，我们可以先计算出后n-1位的所有字母组合，然后再与第1位数字对应的多个字母进行组合，如此递归即可求得最终解。这种解法思路比较直观，需要产生n次函数调用，但每次递归结束需要遍历中间结果（vector），然后与当前数字的多个数字进行组合。
2. 采用回溯法，基于深度优先对所有数字进行遍历。产生约$3^n$次函数调用，但是减少了vector的构造、回收与遍历。

### 代码：
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

