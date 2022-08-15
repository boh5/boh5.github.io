---
title: 数据结构和算法学习笔记（一）复杂度和简单排序算法
last_modified_at: 2022-08-15T23:54+08:00
toc: true
toc_sticky: true
excerpt_separator: <!--more-->
categories:
  - 学习笔记
  - datastructures-algorithms
tags:
  - 数据结构
  - 算法
  - 时间复杂度
  - 空间复杂度
  - 排序算法
  - 位运算
  - 二分查找
---

## 1. 时间复杂度

- 时间复杂度：算法总操作数的表达式中，只要高阶项，不要高阶项的系数，剩下的部分假设为 f(n)，则时间复杂度为 O(f(n))
- O(f(n)) 按照算法可能遇到的**最差**的情况估计
- 如果两个算法复杂度相同，很难区分哪个更好，可以通过测试的方式，确定选择

## 2. 位运算

### 2.1 `^` 异或运算：也可以理解成无进位相加

性质：

- `0 ^ N = N`  `N ^ N = 0`
- 满足交换律、结合律：从**无进位相加**来考虑，每个二进制位的值之和每个数上同位的值的个数(奇偶)有关

因此，交换变量的值：

```python
a = a ^ b  # a = a ^ b, b = b
b = a ^ b  # b = a ^ b ^ b = a, a = a ^ b
a = a ^ b  # a = a ^ b ^ a = b, b = a
```

> a、b 的值可以一样，但不能是同一块内存空间，否则会抹成0，如 a, b 是 list 中相同的 index

Question：
1. [136. 只出现一次的数字](https://leetcode.cn/problems/single-number/)，要求：时间复杂度O(n)，空间复杂度O(1)
2. [260. 只出现一次的数字 III](https://leetcode.cn/problems/single-number-iii/)，要求：时间复杂度O(n)，空间复杂度O(1)
  - 提取一个数二进制位中最右侧的 1(其他位都置为 0): `a & (~a + 1)`

<!--more-->

## 3. 排序算法

### 3.1 选择排序

```python
def selection_sort(arr: List[int]):
    if not arr or len(arr) <= 1:
        return

    for i in range(len(arr)):
        min_index = i
        for j in range(i, len(arr)):
            if arr[j] < arr[min_index]:
                min_index = j
        arr[i], arr[min_index] = arr[min_index], arr[i]
```

时间复杂度：O(n^2)

### 3.2 冒泡排序

```python
def bubble_sort(arr: List[int]):
    if not arr or len(arr) <= 1:
        return

    for i in range(len(arr) - 1, -1, -1):
        for j in range(i):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]

```

时间复杂度：O(n^2)

### 3.3 插入排序

```python
def selection_sort(arr: List[int]):
    if not arr or len(arr) <= 1:
        return

    for i in range(len(arr)):
        cur_num = arr[i]
        while i > 0 and arr[i - 1] > cur_num:
            arr[i] = arr[i - 1]
            i -= 1
        arr[i] = cur_num
```

时间复杂度：O(n^2)；最好的情况（数组已经有序）下，时间复杂度O(n)

## 4. 二分查找

### 4.1 有序数组中，找某个数是否存在

时间复杂度：O(log_2 N)、O(logN)


### 4.2 有序数组中，找 >= 某个数最左侧的位置

找到后还得继续往左找，一个变量记录上次位置，直到找到底

### 4.3 无序数组中，局部最小值

[162. 寻找峰值](https://leetcode.cn/problems/find-peak-element/)

1. 假如索引 0，1 呈下降趋势，索引 n-1，n-2 也呈下降趋势，那么 0 和 n-1 中间必定有局部最小值
2. 中点索引 mid，如果 mid 值小于左边和右边，找到了，返回，否则
3. 假如 mid，mid-1 呈下降趋势，那么 0 和 mid 中间必定有局部最小值；右边同理
4. 二分继续

## 5. 对数器的概念和使用

> 对数器是通过用大量测试数据来验证算法是否正确的一种方式。

流程：
- 有一个你想要测的方法`a`；
- 实现一个绝对正确但是复杂度不好的方法`b`；
- 实现一个随机样本产生器；
- 实现对比算法`a`和`b`的方法；
- 把方法`a`和方法`b`比对多次来验证方法`a`是否正确；
- 如果有一个样本使得比对出错，打印样本分析是哪个方法出错；
- 当样本数量很多时比对测试依然正确，可以确定方法`a`已经正确。

注意：
- 随机产生的样本应该是小数据集，但是要进行多次(10w+)的对比。小数据集是因为方便对比分析，多次比对是要覆盖所有的随机情况。
- 算法b要保持正确性。

## 6. 递归行为及其时间复杂度估计

master 公式，如果调用子问题规模相等，可以用该公式求时间复杂度：

T(N) = a * T(N / b) + O(N ^ d)

- T(N)：母问题的数据量为 N
- T(N / b)：子问题的规模是 N / b
- a：子问题调用次数
- O(N ^ d)：除了子问题调用，剩下的过程的时间复杂度

求解时间复杂度：
1. 如果 log_b a > d: O(N ^(log_b a))
2. 如果 log_b a == d: O((N ^ d) * log N)
3. 如果 log_b a < d: O(N ^ d)
