---
title: leetcode 678. Valid Parenthesis String
author: Keaper
tags: 
categories: 
date: 2017-09-19 17:39:00
---
## 题目
[https://leetcode.com/problems/valid-parenthesis-string/description/](https://leetcode.com/problems/valid-parenthesis-string/description/)
## 题意
Given a string containing only three types of characters: '(', ')' and '*', write a function to check whether this string is valid. We define the validity of a string by these rules:

Any left parenthesis '(' must have a corresponding right parenthesis ')'.
Any right parenthesis ')' must have a corresponding left parenthesis '('.
Left parenthesis '(' must go before the corresponding right parenthesis ')'.
'*' could be treated as a single right parenthesis ')' or a single left parenthesis '(' or an empty string.
An empty string is also valid.

判断系列是否是合法的括号序列，其中'*'可以替代')'或者‘(’或者空。
## 题解
-  暴力求解

对每个'*'都替换为三种情况递归判断，但是会超时。时间复杂度O（N*3^N）。
```cpp

class Solution {
private:
    char place[3] = {'(',')','*'};
private:
    bool valid(string str) {
        int cnt = 0;
        for(int i=0;i<str.size();i++){
            if(str[i]=='(') cnt++;
            else if(str[i]==')') cnt--;
        }
        return cnt==0;
    }
    bool dfs(string s,int index){
        if(index==s.size()){
            return valid(s);
        }else if(s[index]=='*'){
            for(int k=0;k<3;k++){
                s[index] = place[k];
                if(dfs(s,index+1))
                    return true;
            }
        }else{
            return dfs(s,index+1);
        }
    }
public:
    bool checkValidString(string s) {
        return dfs(s,0);
    }
};
```

- DP

设dp[i][j] 为true，当且仅当s[i], s[i+1], ..., s[j]是合适的。
状态转换方程为：如果存在i+1<=k<=j使得s[i][k]为true且s[k+1][j]为true。
```cpp
class Solution {
private:
    vector< vector<int> > dp;
public:
    bool checkValidString(string s) {
        if(s.size()==0) return true;
        dp.resize(s.size(),vector<int>(s.size()));

        for(int i=0;i<s.size();i++)
            if(s[i]=='*') dp[i][i] = true;

        for(int size = 1;size<s.size();size++){
            for(int i=0;i+size<s.size();i++){
                if(s[i]=='('||s[i]=='*'){
                    for(int k=i+1;k<=i+size;k++){
                        if((s[k]==')'||s[k]=='*')&&
                        (k==i+1||dp[i+1][k-1])&&
                        (k==i+size||dp[k+1][i+size]))
                            dp[i][i+size] = true;
                    }
                }
            }
        }
        return dp[0][s.size()-1];
    }
};
```
时间复杂度O(N^3)，空间复杂度O(N^2)。
- 贪心

依次扫描，维护两个值，cmin代表，最少的可能不匹配的左括号，cmax代表最多的可能不匹配的左括号数量。
显然，如果当前字符为‘(’,cmin，cmax都加1；
如果当前字符为‘)’，cmin，cmax都减1，但是cmin至少应该为0；如果cmax小于0，标识右括号数量大于左括号，不匹配。
如果当前字符为‘*’，如果替换为‘(’,cmax加1，如果替换为')'，cmin减1，同样至少为0。
最后判断cmin是否为0.
```cpp
class Solution {
public:
    bool checkValidString(string s) {
        int cmin = 0,cmax = 0;
        for(int i=0;i<s.size();i++){
            if(s[i]=='('){
                cmin++;
                cmax++;
            }
            else if(s[i]==')'){
                cmin = max(cmin-1,0);
                cmax--;
            }else{
                cmax++; //*=left
                cmin=max(cmin-1,0); //*=right
            }
            if(cmax<0) return false;
        }
        return cmin==0;
    }
};
```
时间复杂度O(N)，空间复杂度O(1).
