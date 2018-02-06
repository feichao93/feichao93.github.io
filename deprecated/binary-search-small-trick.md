---
title: 二分搜索处理下标的技巧
date: 2016-08-18 21:33:55
tags: 二分法
---

```java
// Java二分法.
// array已按从大到小排好序, 且array中无重复元素.
// start和end是inclusive的
static int binarySearch(int[] array, int start, int end, int target) {
    while (start <= end) {
        int median = (start + end) / 2;
        if (array[median] < target) {
            start = median + 1;
        } else if (array[median] > target) {
            end = median - 1;
        } else {
            return median; // A
        }
    }
    return -1; // B
}
```
二分法本身不复杂, 但是实际的编码过程中, 根据搜索条件的不同, 我们处理下标的方式也不同. 一般应用二分法, 会涉及到下面五种情况.
1. 找出`array`中**等于**`target`的数字的下标. 如果不存在这样的数字, 则返回`-1`
2. 找出`array`中第一个**大于**`target`的数字的下标. 如果不存在这样的数字, 则返回`-1`
3. 找出`array`中第一个**大于等于**`target`的数字的下标. 如果不存在这样的数字, 则返回`-1`
4. 找出`array`中最后一个**小于**`target`的数字的下标. 如果不存在这样的数字, 则返回`array.length`
5. 找出`array`中最后一个**小于等于**`target`的数字的下标. 如果不存在这样的数字, 则返回`array.length`

这五种情况分别对应 `==` / `>` / `>=` / `<` / `<=` 这五种不同的比较.

针对不同的情况, 我们在A / B两处返回的值也应当不同, 五种情况的差异细微, 如果没有好的区分方法, 每次都要具体情况具体分析, 这样实现二分即繁杂又容易出错. 下面我们从返回值的角度入手, 介绍一个可以快速区分这五种情况并写出相应代码的方法.

## A处的情况处理
如果`target`在`array`中, 则函数将在A处返回, 此时`array[median] = target`. 那么上述五种情况的对应写法如下:
1. **等于**: `median`
2. **大于**: `median+1`
3. **大于等于**: `median`
4. **小于**: `median-1`
5. **小于等于**: `median`

## B处的情况处理
### 14行满足 `start = end + 1`
如果`target`不在`array`中, 则函数将在B处(14行)返回. 在14行处, 此时有`start > end`. 在函数没有跳出while之前, 总是有`start <= median <= end`, 而`start`每次变化之后只会比`median`大`1`, 可知`start`每次变化后也最多只会比`end`大`1`(同理`end`每次变化后最多比`start`小1). 跳出循环后, `start > end`. 因为两者最多相差`1`, 故在14行我们有`start = end + 1`

### 14行满足 `array[end] < target < array[start]`
在跳出while的最后一次迭代时, 我们有`start == end == median`(只有这样跳出循环的时候才能满足`start = end + 1`) 这里解释不清楚了, 算了以后再说吧(￣3￣)

有了上面这两条, 我们就可以得到下面的结论: `end`比`start`小`1`, 且如果我们要将`target`插入到数组中, 则`target`应该插在`end`和`start`之间.

下面的流程图中展示了二分搜索的各个步骤. 注意最后一步中, `end`, `start`和`target`插入位置的关系.

![二分法流程图][1]

有了上面的结论, 那么B处的5种返回情况也就很容易理解了:
1. **等于**: `-1`
2. **大于**: `start`
3. **大于等于**: `start`
4. **小于**: `end`
5. **小于等于**: `end`

## 数组存在重复元素的情况
(未完待续)

[1]:  ./static/binary-search.jpg
