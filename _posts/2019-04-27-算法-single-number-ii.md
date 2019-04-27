---
layout: post
title: LeetCode-single-number-ii
author: liuzhimi
---
-----
LeetCode-single-number-ii
### 题目
**题目描述**
Given an array of integers, every element appears three times except for one. Find that single one.
**Note**: 
Your algorithm should have a linear runtime complexity. Could you implement it without using extra memory?

### 解题思路
用二进制去模拟三进制，设a是高位，b是低位
三个相同的数(b为1或0)相加,记这个数为c

1.b=1, c+c+c => b=0

2.b=0, c+c+c => b=0

A的卡诺图(状态转移)

| AB C | 0 | 1|
| :-: | :-: | :-: |
| 00 | 0 | 0 |
| 01 | 0 | 1 |
| 10 | 1 | 0 |

B的卡诺图(状态转移)

| AB C | 0 | 1|
| :-: | :-: | :-: |
| 00 | 0 | 1 |
| 01 | 1 | 0 |
| 10 | 0 | 0 |

总的卡诺图(状态转移)

|此时状态| ta | tb | c | a | b |
|:-:| :-: | :-: | :-: | :-: | :-: |
|已加上0个c| 0 | 0 | 0 | 0 | 0|
|已加上0个c| 0 | 0 | 1 | 0 | 1|
|已加上1个c| 0 | 1 | 0 | 0 | 1|
|已加上1个c| 0 | 1 | 1 | 1 | 0|
|已加上2个c| 1 | 0 | 1 | 1 | 0|
|已加上2个c| 1 | 0 | 1 | 0 | 0|


因此可以得出：
a = (ta & ~tb & ~c)  |  (~ta & tb & c)
b = ~ta & ((~tb & c) | (tb & ~c))

**Note**：
数组[2,1,3,2,3,3,2]和数组[2,2,2,3,3,3,1]运算结果是一样的，因为位运算是满足交换律的。

### Code
```
public class Solution {
    public int singleNumber(int[] A) {
        int a = 0;
        int b = 0;
        for(int c : A){
            int ta, tb;
            ta = a;
            tb = b;
            a = (ta&~tb&~c) | (~ta&tb&c);
            b = ~ta&((~tb&c)|(tb&~c));
        }
        return a|b;
    }
}
```