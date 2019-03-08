---
title: leetcode 6. ZigZag Conversion
tags:
categories:
date: 2017-07-02 18:47:00
---
## 题目链接：
[https://leetcode.com/problems/zigzag-conversion/#/description](https://leetcode.com/problems/zigzag-conversion/#/description)
## 题意：
给一个字符串，要求将字符串排列成锯齿状，然后按行从左到右输出。如下图，原来的字符串顺序为： BFGAHIDJKCLME，按行读就是BDEFIJMGHKLAC。
![图示](https://blog-picture.nos-eastchina1.126.net/picture0004.png)

## 题解：
找规律即可，按行来看相邻两个点的距离分为两个，假设为a和b，第 i 行为[2*(n-i-1)，2*i]，第一行相当于b为0，第二行相当于a为0，距离为零表示两点重合，不考虑。依次输出。
## 代码：
```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution
{
public:
    string convert(string s, int numRows)
    {
        if(numRows==1) return s;
        string ans="";
        for(int i=0; i<numRows; i++)
        {
            int a=2*(numRows-i-1),b=2*i;
            for(int j=i; j<s.size();)
            {
                if(a){
                    ans+=s[j];
                    j+=a;
                }
                if(j>=s.size()) break;
                if(b){
                    ans+=s[j];
                    j+=b;
                }
            }
        }
        return ans;
    }
};
int main()
{
    Solution sol;
    string str;
    int n;
    while(cin>>str>>n){
        cout<<sol.convert(str,n)<<endl;
    }
    return 0;
}
```