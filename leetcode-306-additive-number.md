---
title: Leetcode-306 Additive Number
data: 2016-09-29 00:00:00
tags: leetcode
---

刷了那么久的leetcode, 今天终于靠自己写的算法打败了99%的提交. 先来看看题意([原题点我](https://leetcode.com/problems/additive-number/)).

Additive number is a string whose digits can form additive sequence.

A valid additive sequence should contain **at least** three numbers. Except for the first two numbers, each subsequent number in the sequence must be the sum of the preceding two.

For example:

`"112358"` is an additive number because the digits can form an additive sequence: `1, 1, 2, 3, 5, 8`.

`1 + 1 = 2, 1 + 2 = 3, 2 + 3 = 5, 3 + 5 = 8`

`"199100199"` is also an additive number, the additive sequence is: `1, 99, 100, 199`.

`1 + 99 = 100, 99 + 100 = 199`

**Note:** Numbers in the additive sequence **cannot** have leading zeros, so sequence `1, 2, 03` or `1, 02, 3` is invalid.

Given a string containing only digits `'0'-'9'`, write a function to determine if it's an additive number.

**Follow up:**
How would you handle overflow for very large input integers?

## 解题思路
拿到一个字符串, 一旦我们确定了`additive number`序列的前两项, 就可以确定序列后面的数字. 设第一个数长度为`firstLen`, 第二个数长度为`secondLen`, 不断计算后面的数字, 一旦发现数字与给定的字符串不匹配, 则返回`false`. 如果`firstLen`和`secondLen`对应的序列与字符串相匹配, 返回`true`.

### 优化关键点
注: `num.substring`即为`String#substring`.

下面代码中主要是用`checksum`函数来判断字符串中某一子串(对应`number1`)加上与其相邻的子串(对应`number2`)的结果是否匹配`number2`后面的字符串.

`checksum(int x, int y, int z)`用来检测`num.subtring(x, y)`(记作`number1`)和`num.substring(y, z)`(记作`number2`)相加之后的结果是否为`num.subtring(z)`的前缀. 如果不为前缀, 则表示匹配失败, 返回`-1`. 如果匹配成功, 返回`w`, 且`num.substring(z, w)`为`number1`和`number2`的和.

`number1`和`number2`相加时. 如果这两个数位数相等且在最高位相加的时候发生进位, 则结果的长度比`number1`的长度大1. 否则, 结果的长度和`number1`/`number2`中较长的数的长度相等.
取`size = Math.max(number1.length, number2.length)`, 则结果的长度为`size`或`size + 1`. 我们分别判断这两种情况即可.

判断`number1`和`number2`相加是否匹配字符串时, 我们从个位开始, 将其相加, 判断是否与结果的个位(因为知道了结果可能的长度, 所以也就知道了结果的可能的个位)匹配. 然后再判断十位, 然后百位... 注意要考虑进位问题, 可以用变量`carry`来记录进位情况.

## AC代码
```java
// 代码还有些粗糙. 可以稍微改进一下.
public class Solution {
    public boolean isAdditiveNumber(String num) {
        char[] word = num.toCharArray();
        int maxLen = word.length / 2; // 单个数字的最大长度
        for (int firstLen = 1; firstLen <= maxLen; firstLen++) {
            if (word[0] == '0' && firstLen > 1) {
                break; // 1. 考虑数字的开头的0是不可法, 跳过这些情况
            }
            for (int secondLen = 1; secondLen <= maxLen && firstLen + secondLen < word.length; secondLen++) {
                if (word[firstLen] == '0' && secondLen > 1) {
                    break; // 2. 和1处相同
                }
                int x = 0;
                int y = firstLen;
                int z = firstLen + secondLen;
                while (z < word.length) {
                    int next = checksum(word, x, y, z);
                    if (next == -1) {
                        break;
                    }
                    // 3. 正确设置x,y,z的值, 用于进行下一处的判断
                    // (x, y, z) = (y, z, next)
                    x = y;
                    y = z;
                    z = next;
                }
                if (z == word.length) {
                    return true; // 匹配到了word的最后, 说明完成了所有的匹配
                }
            }
        }
        // 没有一种的firstLen/secondLen的组合可以得到完整的匹配, 则返回false
        return false;
    }

    // 判断 word[x...y] + word[y...z]的结果是否为 word[z...]的前缀
    int checksum(char[] word, int x, int y, int z) {
        int c1 = y - x;
        int c2 = z - y;
        int size = Math.max(c1, c2);
        boolean mayCarry = z + size < word.length; // 有可能得到size+1的情况
        if (mayCarry) {
            int w = z + size + 1; // 结果个位下标再加上1
            int carry = 0; // 记录进位情况
            int t = 1; // 循环变量. t=1时判断个位, t=2时判断十位, t=3时判断百位...
            while (t <= size) {
                int m = (y - t >= x) ? (word[y - t] - '0') : 0; // number1的t位
                int n = (z - t >= y) ? word[z - t] - '0' : 0; // number2的t位
                int h = word[w - t] - '0'; // 结果的t位

                int sum = m + n + carry;
                if (sum % 10 != h) {
                    break;
                }
                carry = sum / 10;
                t++;
            }
            if (t == size + 1 && carry == 1 && (word[z] - '0') == 1) {
                return z + t;
            }
        }
        // 不进位 结果为size位
        int w = z + size;
        if (w > word.length) {
            return -1;
        }
        int carry = 0;
        int t = 1;
        while (t <= size) {
            int m = (y - t >= x) ? (word[y - t] - '0') : 0;
            int n = (z - t >= y) ? (word[z - t] - '0') : 0;
            int h = word[w - t] - '0';

            int sum = m + n + carry;
            if (sum % 10 != h) {
                return -1;
            }
            carry = sum / 10;
            t++;
        }
        if (carry == 0) {
            return z + size;
        }
        return -1;
    }
}
```
