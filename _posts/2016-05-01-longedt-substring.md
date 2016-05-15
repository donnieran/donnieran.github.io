---
layout:     post
title:      "Longest Substring Without Repeating Characters"
#subtitle:   "Everything is a file"
date:       2016-05-01 21:19:00
author:     "Don"
header-img: "img/black.jpg"
tags:
    - LeetCode
    - 算法	
---

## 题目

Given a string, find the length of the longest substring without repeating characters.

**Examples:**

Given "abcabcbb", the answer is "abc", which the length is 3.

Given "bbbbb", the answer is "b", with the length of 1.

Given "pwwkew", the answer is "wke", with the length of 3. Note that the answer must be a substring, "pwke" is a subsequence and not a substring.

Subscribe to see which companies asked this question

**即为：**给定一个字符串，输出此字符串中连续的不含重复元素的子字符串的最大长度。

## 分析

此题目可用**贪婪算法**。即顺序遍历此字符串，在每个元素位置取得符合题目要求的往前的最大子字符串长度，取最大值。所以关键点在于取得子字符串的方法。

## C语言实现

{% highlight rouge %}

int lengthOfLongestSubstring(char* s) {
    int i = 0;
    int last[128];			 /* hash表，记录字符上一次出现的位置 */
    int len = 0;
    int subStringStart = -1; /* -1:假设字符串中只有一个字符情况, 该位置+1为子字符串的开始位置 */
    memset(last, -1, sizeof(last));
    for(i = 0; s[i] != '\0'; ++i)
    {
        if(last[s[i]] > subStringStart )
        {
            subStringStart = last[s[i]];	/* 子字符串开始的位置 */
        }
        
     
        if(len < i - subStringStart )
        {
            len = i - subStringStart;  		/* 取最大值*/  
        }
          
        last[s[i]] = i;
    }
    return len;
    
}

{% endhighlight %}

