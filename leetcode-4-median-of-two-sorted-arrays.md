---
title: Leetcode-4 Median of Two Sorted Arrays
data: 2017-02-01 23:33:00
tags: leetcode
---

[原题链接](https://leetcode.com/problems/median-of-two-sorted-arrays/)

此题使用二分法，难点在于如何去构造二分规则。我们记这两个数组为$A$ 和 $B$，记结果为$x$。

## 解题思路

假设中位数$x$位于$A[i]$与$A[i+1]$之间，即有$A[i] \leq x < A[i+1]$。假设中位数$x$位于$B[j]$与$B[j+1]$之间，即有$B[j] \leq x < B[j+1]$。下面是一个比较简单的例子，从例子中我们可以看出$i$和$j$的含义。
$$
i = 2\\
\begin{align*}
& A\quad 1\ 3\ \underline{5\ 7\ }9\\
& B\quad \quad 4\ \underline{6\ 8\ }10\ 12\\
\end{align*}\\
j = 1\\
\mbox{上述两个数组的中位数为} \frac{6+7}{2}=6.5
$$

A数组中比中位数$x​$小的数字有$(i+1)​$个，B数组中有$(j+1)​$个，根据中位数的定义，小于中位数的数字数量应该为总数的一半，故有条件$①​$:
$$
(i + 1) + (j + 1) = \frac{M+N}{2}\cdots\cdots①
$$
根据条件$①$，我们也就能知道$j$关于$i$的表达式了。

根据$A[i] \leq x < A[i+1]$和$B[j]<= x < B[j+1]$我们也能得到条件$②$和条件$③$：
$$
A[i] \leq B[j+1]\cdots\cdots②\\
B[j] \leq A[i+1]\cdots\cdots③
$$
根据条件$①②③$，我们可以用二分法来得到$i$的值：

## 二分法得到$i$的值的代码(JavaScript)

```javascript
let min = 0
let max = M - 1 
// 这里min/max的值设置比较简单 还需要考虑一些edge cases
while (min <= max) {
  const i = (min + max) / 2
  // 注意这里(M + N)如果为奇数的话 除以2之后j会比正常值小0.5
  // 这也是中位数是 c 而不是 b 的原因（b/c的含义下面会说明）
  const j = (M + N) / 2 - 2 - i
  const condition2 = A[i] <= B[j+1]
  const condition3 = B[j] <= A[i+1]
  if (condition2 && condition3) { // 找到了正确的i值
    break;
  } else if (!condition2 && condition3) {
    // i的值太大了
    max = i - 1;
  } else if (condition2 && !condition3) {
    // i的值太小了
    min = i + 1;
  } // else 两个条件都不成立  这种情况不可能发生
}
const i = (min + max) / 2 // 循环结束时计算得到i的值
```

使用二分法得到$i$的值之后，我们可以得到四个数字：$A[i],\ A[i+1],\ B[j],\ B[j+1]$，将这四个数字排序，得到$a\ b\ c\ d$。我们知道**$a\ b\ c\ d$这四个数字位于两个数组中间，中位数可以由这四个数字得出**。在不考虑edge cases的情况下，如果两个数组共有偶数个数字，则中位数为$\frac{b+c}{2}$，否则中位数即为$c$。

以上算法复杂度为$log(M)$，通过比较两个数组长度大小并交换数组的话，可以得到$log(min\{M,N\})$的算法。

上述解法中还需要考虑很多的edge cases，例如：

1. 空数组或是数组中只有一个元素
2. 像`A:1,2,3   B:4,5,6`这样$i$或$j$在数组两端的情况

解决思路：在A、B数组的开头插入一个`Integer.MIN_VALUE`，在两个数组的末尾插入一个`Integer.MAX_VALUE`。插入这样的值既不影响结果，又可以避免上面提到的两种edge cases。（这只是解决的思路，实际编程的时候可以参考下面java代码的做法）。

## AC代码(java)

```java
import java.util.Arrays;

public class Solution {
    public double findMedianSortedArrays(int[] A, int[] B) {
        int M = A.length;
        int N = B.length;
        if (M > N) { // 交换两个数组
            return findMedianSortedArrays(B, A);
        }
        if (M == 0) { // 考虑空数组的情况
            return median(B);
        }
        // 这里min的初始值为-1.
        // 是因为我们在数组起始处"插入"了一个Integer.MIN_VALUE
        int min = -1;
        int max = M - 1;
        int i = (min + max) / 2;
        int j = (M + N) / 2 - 2 - i;
        while (min <= max) {
            i = (min + max) / 2;
            j = (M + N) / 2 - 2 - i;

            boolean condition2 = get(A, i) <= get(B, j + 1);
            boolean condition3 = get(B, j) <= get(A, i + 1);
            if (condition2 && condition3) {
                break;
            } else if (!condition2) {
                max = i - 1;
            } else { // condition2 && !condition3
                min = i + 1;
            }
        }

        int[] array = {
                get(A, i),
                get(A, i + 1),
                get(B, j),
                get(B, j + 1)
        };
        Arrays.sort(array);
        if ((M + N) % 2 == 0) {
            return (array[1] + array[2]) / 2.0;
        } else {
            return array[2];
        }
    }

    private static double median(int[] array) {
        int size = array.length;
        if (size % 2 == 0) {
            return (array[size / 2] + array[size / 2 - 1]) / 2.0;
        } else {
            return array[size / 2];
        }
    }

    private static int get(int[] array, int index) {
        if (index < 0) {
            return Integer.MIN_VALUE;
        } else if (index >= array.length) {
            return Integer.MAX_VALUE;
        } else {
            return array[index];
        }
    }
}
```
