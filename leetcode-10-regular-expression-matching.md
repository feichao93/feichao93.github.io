---
title: Leetcode-10 Regular Expression Matching
date: 2017-02-05 17:09:00
tags: leetcode
---

[原题链接](https://leetcode.com/problems/regular-expression-matching/)

题目中给的模式串中包含星号`*`，星号本身并不匹配字符，其作用是修饰前一个模式字符。如果我们直接使用模式字符串，一个匹配项将对应一个字符或两个字符。为了处理该情况，我们定义了`Entry`类，一个`Entry`实例表示一个匹配项。`Entry#matchChar`也简化了单个字符的匹配问题。代码中*17-29行*将模式字符串转换为`Entry`列表。

记字符串的长度为$M$，匹配项列表长度为$N$，应用动态规划，我们可以得到$O(M*N)$的算法。

我们定义二维数组`matrix`，其中`matrix[i][j]`表示字符串的前`i`个字符和前`j`个entry是否匹配。`matrix[M][N]`表示整个字符串与整个模式串是否匹配，即为要求的结果。

## 递推关系

该动态规划中共有三种不同的递推情况。当我们要求`matrix[i][j]`时，只要满足以下三种情况中的一种，我们就可以将`matrix[i][j]`置为true，否则需置为false。

#### 第一种情况：单个字符与单个entry匹配 *代码56行*

如果前`i-1`个字符与前`j-1`个entry相匹配，且`str[i-1]`和`entries[j-1]`相匹配，则我们得到：前`i`个字符和前`j`个entry匹配。例如：
$$
\begin{align}
字符串: &\quad \underline{a}\ \boxed{b}\ c\ c\ c\ c\ e\ f\ g \\
entries: &\quad \underline{a}\ \boxed{b}\ c\!*\ d\!* e\ f\ g \\
&\quad \Downarrow \\
字符串: &\quad \underline{a\ b}\ c\  c\ c\ c\ e\ f\ g \\
entries: &\quad \underline{a\ b}\ c*\ d* e\ f\ g \\
\end{align} \\
说明：\underline{带下划线}的部分表示字符串或\mbox{entries}相匹配的部分 \\
\boxed{框起来}的字符或\mbox{entry}表示本次匹配的内容
$$
在上面的例子中，`i`的值为2，`j`的值为2。根据**`matrix[1][1]`为true** 且 **`str[1]`与`entries[1]`相匹配**，我们得到：`matrix[2][2]`为true。

#### 第二种情况：字符与上一个带星号的entry匹配 *代码58行*

如果前`i-1`个字符与前`j`个entry相匹配，且`str[i-1]`匹配`entries[j-1]`的星号部分，则我们得到：前`i`个字符与前`j`个entry匹配。例如：
$$
\begin{align}
字符串: &\quad \underline{a\ b\ c\ c}\ \boxed{c}\ c\ e\ f\ g \\
entries: &\quad \underline{a\ b\ \boxed{c*}}\ d* e\ f\ g \\
&\quad \Downarrow \\
字符串: &\quad \underline{a\ b\ c\  c\ c}\ c\ e\ f\ g \\
entries: &\quad \underline{a\ b\ c*}\ d* e\ f\ g \\
\end{align}
$$
在上面的例子中，`i`的值为5，`j`的值为3。根据**`matrix[4][3]`为true** 且 **`str[4]`与`entries[2]`相匹配** 以及 **`entries[2]`带星号**，我们得到：`matrix[5][3]`为true。

#### 第三种情况：跳过一个带星号的entry *代码60行*

如果前`i`个字符与前`j-1`个entry相匹配，且`entries[j-1]`带星号，则我们得到：前`i`个字符与前`j`个entry匹配。例如：
$$
\begin{align}
字符串: &\quad \underline{a\ b\ c\ c\ c\ c}\ e\ f\ g \\
entries: &\quad \underline{a\ b\ c*}\ \boxed{d*}\ e\ f\ g \\
&\quad \Downarrow \\
字符串: &\quad \underline{a\ b\ c\  c\ c\ c}\ e\ f\ g \\
entries: &\quad \underline{a\ b\ c*\ d*}\ e\ f\ g \\
\end{align}
$$
在上面的例子中，`i`的值为5，`j`的值为4。根据**`matrix[5][3]`为true** 且 **`entries[3]`带星号**，我们得到：`matrix[5][4]`为true。



上面就是三种不同的情况，第一种情况1个字符和1个entry相对应，第二种情况1个字符和0个entry相对应，第三种情况0个字符和1个entry相对应。所有的匹配情况都可以由这三种基本情况演化而来。

## `matrix`初始情况

矩阵的左上角为`matrix[0][0]`，因为**0个字符和0个entry相匹配**，所以`matrix[0][0]`为true。 *代码36行*

矩阵的第一行（不包括左上角）对应**0个字符与`j`个entry是否匹配**。因为**0个字符匹配连续`n`个带星号的entry**，所以矩阵第一行应为**`n`个true，`N - n`个false**。 *代码40-46行*

矩阵的第一列（不包括左上角）对应**`i`个字符与0个entry是否匹配**。显然，非空字符串和空的模式串一定不匹配的，所以第一列**均为false**。 *代码47行*

## AC代码(Java)

```java
import java.util.ArrayList;
import java.util.List;

public class Solution {
  private static class Entry {
    boolean any;
    boolean asterisk;
    char ch;

    boolean matchChar(char c) {
      return any || c == ch;
    }
  }

  public boolean isMatch(String str, String p) {
    char[] pattern = p.toCharArray();
    List<Entry> entries = new ArrayList<>();
    for (int index = 0; index < pattern.length; index++) {
      Entry entry = new Entry();
      entry.ch = pattern[index];
      if (pattern[index] == '.') {
        entry.any = true;
      }
      if (index + 1 < pattern.length && pattern[index + 1] == '*') {
        entry.asterisk = true;
        index++;
      }
      entries.add(entry);
    }
    int rows = str.length() + 1;
    int cols = entries.size() + 1;
    // matrix[i][j]的含义为: 前i个字符和前j个entry是否匹配
    boolean[][] matrix = new boolean[rows][cols];
	// 设定矩阵的初始情况
    // 设定左上角 0个字符匹配0个entry, 所以设置为true
    matrix[0][0] = true;
    // 设定第一行
    // 0个字符匹配连续n个带星号的entry
    // 但只要出现一个不带星号的entry, 后续的entry就都为不匹配
    for (int j = 1; j < cols; j++) {
      if (entries.get(j - 1).asterisk) {
        matrix[0][j] = true;
      } else {
        break;
      }
    }
    // 第一列均为false, 就不需要额外的设置了

    // 开始计算matrix中的每一项
    for (int i = 1; i < rows; i++) {
      for (int j = 1; j < cols; j++) {
        char c = str.charAt(i - 1);
        Entry entry = entries.get(j - 1);

        // 第一种情况 单个字符与单个entry匹配
        boolean option1 = matrix[i - 1][j - 1] && entry.matchChar(c);
        // 第二种情况 字符与上一个带星号的entry匹配
        boolean option2 = matrix[i - 1][j] && entry.matchChar(c) && entry.asterisk;
        // 第三种情况 跳过一个带星号的entry
        boolean option3 = matrix[i][j - 1] && entry.asterisk;

        matrix[i][j] = option1 || option2 || option3;
      }
    }

    return matrix[rows - 1][cols - 1];
  }
}
```