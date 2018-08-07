title: leetcode 680. Valid Palindrome II
author: Keaper
tags: []
categories:
  - leetcode
date: 2017-09-19 14:44:00
---
## 题目
[https://leetcode.com/problems/valid-palindrome-ii/description/](https://leetcode.com/problems/valid-palindrome-ii/description/)
## 题意
Given a non-empty string `s`, you may delete at most one character. Judge whether you can make it a palindrome.

给出一一个非空的字符串，判断是否能够删除0个或者一个字符使得字符串变为回文。

## 题解
1. 暴力。判断原串是否回文，如果不是，遍历每个字符，判断删掉该字符能否构成回文。时间复杂度O(N^2) ,超时。
2. 因为最多只能删掉一个字符，所以，先从两端开始判断，如果相等向中间逼近，如果不等，则跳过一个字符（左边或者右边），判断余下的串是否回文。

```cpp
class Solution
{
public:
    bool validPalindrome(string s)
    {
        int i=0,j=s.size()-1;
        while(i<=j&&s[i]==s[j])
        {
            i++;
            j--;
        }
        if(i>j) return true; //no delete

        int ni = i,nj = j-1;
        while(ni<=nj&&s[ni]==s[nj])
        {
            ni++;
            nj--;
        }
        if(ni>nj) return true; //delete j
        ni = i+1,nj = j;
        while(ni<=nj&&s[ni]==s[nj])
        {
            ni++;
            nj--;
        }
        if(ni>nj) return true; //delete i
        return false;
    }
};

```