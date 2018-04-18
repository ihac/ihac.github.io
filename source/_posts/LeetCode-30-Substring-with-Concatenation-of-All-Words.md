---
title: '[LeetCode] 30. Substring with Concatenation of All Words'
date: 2018-04-18 20:43:49
tags: ['Algorithm', 'C/C++']
categories:
- [Algorithm, LeetCode]
visible: hide
---

### 题目：

    You are given a string, s, and a list of words, words, that are all of the same length.
    Find all starting indices of substring(s) in s that is a concatenation of each word in words exactly once and without any intervening characters.

### 标签：
`Hash Table`, `Two Pointers`, `String`

### 分析：
本题和第3题比较相似，都是采用滑动窗口的思路来求解：维护一前一后两个指针，指针包含的区域即为窗口，始终保证窗口内部的字符串与题目要求不冲突；尽可能地拉伸窗口（即将后指针右移），当窗口内部字符串与题目要求冲突时，通过缩小窗口（右移前指针）来解决冲突。
具体地：
- 首先统计`words`内各个word的出现次数，生成频率表。
- 遍历整个字符串，
    - 当word在频率表中可以查到时，将其加入滑动窗口，并累计滑动窗口内每个word的出现次数，如果超过对应频率表中的出现次数，缩小滑动窗口（右移前指针直至该word被移出）；
    - 如果word在频率表中不存在，说明这是一个invalid word，滑动窗口归零，将前指针重置为当前位置，继续遍历；
- 当滑动窗口大小等于`words`内所有word的长度之和时，得到题解，存下前指针的位置，然后将前指针右移一格，继续遍历。

需要注意的是，由于所有word等长，且没有交叉字符，因此只需要wordLen（单词长度）轮遍历即可完成任务。

假设word长度为$l$，字符串长度为$n$，计算复杂度为$O(ln)$

### 代码：

``` c++
class Solution {
public:
    vector<int> findSubstring(string s, vector<string>& words) {
        if (words.size() == 0) return vector<int>{};
        vector<int> res;
        // count the occurrence number of each word
        map<string, int> countMap;
        for (int i = 0; i < (int)words.size(); i++) {
            if (countMap.find(words[i]) == countMap.end())
                countMap[words[i]] = 1;
            else
                countMap[words[i]]++;
        }

        int wordLen = words[0].length(), totalLen = wordLen * words.size();
        /*
         * All words are of the same length, and
         * there are no intervening characters.
         */
        for (int i = 0; i < wordLen; i++) {
            // sliding window
            int start = i, curr = i;
            map<string, int> newCount;
            while (start + totalLen <= (int)s.length()) {
                string ss = s.substr(curr, wordLen);
                // current word is invalid
                if (countMap.find(ss) == countMap.end()) {
                    newCount.clear();
                    start = curr + wordLen;
                }
                else if (newCount.find(ss) != newCount.end()) {
                    newCount[ss]++;
                    // the occurrence number of current word is larger than needed,
                    // slide the `start` pointer until one current word is skipped.
                    if (newCount[ss] > countMap[ss]) {
                        while (start < curr) {
                            string _ss = s.substr(start, wordLen);
                            newCount[_ss]--;
                            start += wordLen;
                            if (_ss == ss) break;
                        }
                    }
                }
                else
                    newCount[ss] = 1;
                curr += wordLen;
                if (curr - start == totalLen) {
                    res.push_back(start);
                    string _ss = s.substr(start, wordLen);
                    newCount[_ss]--;
                    start += wordLen;
                }
            }
        }
        return res;
    }
};

```
