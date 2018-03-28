---
title: '[LeetCode] 28. Implement strStr()'
date: 2018-03-28 10:38:11
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    Implement strStr().
    Return the index of the first occurrence of needle in haystack, or -1 if needle is not part of haystack.

    Example 1:
    Input: haystack = "hello", needle = "ll"
    Output: 2

    Example 2:
    Input: haystack = "aaaaa", needle = "bba"
    Output: -1

### 标签：
`Two Pointers`, `String`, `KMP`

### 分析：
注意edge case的存在，比如当needle为`""`时，结果应该是0，而非-1。
最简单的做法是，遍历haystack，当出现与needle[0]匹配的字符时，进入匹配循环，如果匹配成功，即可返回当前索引；如果匹配失败，继续遍历。这种算法在最差情况下的复杂度为$O(MN)$（假设needle、haystack长度分别为$m$、$n$）。具体的改进方法有两种：
1. 使用双指针：当找到与needle[0]匹配的字符时，使用两个指针分别从needle头部、尾部同时匹配，当指针出现交叉时，匹配结束。这种方法相较于原始方法，在某些情况下有很大改进，不妨试想haystack=`"111...11"`、needle=`"111..12"`的场景。
2. KMP算法：注意到，无论是原始方法，还是改进的双指针算法，它们都存在着“信息浪费”的情况，即匹配过程中，我们其实已经对haystack进行了一段距离的遍历，但是匹配结束后，我们并没有对遍历结果加以利用，而是重新开始了新的遍历+匹配。KMP算法很巧妙地利用了prefix=suffix的特征，允许我们在匹配结束后，省去一段距离的遍历。具体算法原理在之后的post中会详细说明。

### 代码：

``` c++
class Solution {
public:
    // using two pointers
    int strStr(string haystack, string needle) {
        if (needle.length() > haystack.length()) return -1;
        if (needle == "") return 0;
        int len = needle.length(), start, end;
        for (int i = 0; i < (int)haystack.length(); i++) {
            if (needle[0] == haystack[i]) {
                start = i+1; end = i + len - 1; // start matching from both sides.
                if (end >= (int)haystack.length()) break;
                while (start <= end) {
                    if (haystack[start] != needle[start-i] ||
                        haystack[end] != needle[end-i])
                        break;
                    start++; end--;
                }
                if (start > end) return i;
            }
        }
        return -1;
    }
};
```
