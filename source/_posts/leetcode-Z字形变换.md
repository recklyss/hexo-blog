---
title: 'leetcode:Z字形变换'
date: 2019-04-13 19:43:02
categories: [算法题解]
tags: [leetcode,算法题解]
---
#### 题目如下

将一个给定字符串根据给定的行数，以从上往下、从左到右进行 Z 字形排列。

比如输入字符串为 "LEETCODEISHIRING" 行数为 3 时，排列如下：

```$xslt
L   C   I   R
E T O E S I I G
E   D   H   N
```

之后，你的输出需要从左往右逐行读取，产生出一个新的字符串，比如："LCIRETOESIIGEDHN"。
<!--more-->
请你实现这个将字符串进行指定行数变换的函数：

```
string convert(string s, int numRows);
```



示例 1:

```
输入: s = "LEETCODEISHIRING", numRows = 3
输出: "LCIRETOESIIGEDHN"
```

示例 2:

```
输入: s = "LEETCODEISHIRING", numRows = 4
输出: "LDREOEIIECIHNTSG"
```

解释:

```
L     D     R
E   O E   I I
E C   I H   N
T     S     G
```

#### 解题思路

拿到这个题目，第一时间就可以想到，根据题中图示构造二维数组，先将数据按照相应的样子存储进去，最后再从数组中按行取出，但是这样会有占用更多内存空间的风险。所以，我这边还思考了第二种解法：就是直接根据规律计算出下一个要输出的字符的下标，直接输出即可，无需再创建多余的二维数组。

- 第一种解法：构造二维数组

构造二位数组最主要的就是计算出这个二维数组有多少列，列数有了，按照Z型规律将原字符串塞进去就行了，计算列数代码如下

```
private int getColNum(String s, int n) {
        int x = s.length() / (2 * n - 2);
        int y = s.length() % (2 * n - 2);
        int l = x + 1 + x * (n - 2);
        if (y >= n) {
            l = l + 1 + y % n;
        }
        return l;
    }
```

- 第二种解法：计算下一个要输出的字符的下标
直接看github代码吧：[点这里](https://github.com/Fatezhang/DataStructureAndAlgorithm/tree/master/Algorithm/src/main/java/Alogrithm/Alogrithm/ZigZagConversion)
