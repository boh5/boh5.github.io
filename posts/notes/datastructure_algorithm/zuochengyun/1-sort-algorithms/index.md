# 数据结构和算法学习笔记（一）排序算法


## 1. 复杂度和简单排序算法

### 1.1 时间复杂度

- 时间复杂度：算法总操作数的表达式中，只要高阶项，不要高阶项的系数，剩下的部分假设为 $f(n)$，则时间复杂度为 $O(f(n))$
- $O(f(n))$ 按照算法可能遇到的**最差**的情况估计
- 如果两个算法复杂度相同，很难区分哪个更好，可以通过测试的方式，确定选择

### 1.2 位运算

#### 1.2.1 `^` 异或运算：也可以理解成无进位相加

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
1. [136. 只出现一次的数字](https://leetcode.cn/problems/single-number/)，要求：时间复杂度$O(n)$，空间复杂度$O(1)$
2. [260. 只出现一次的数字 III](https://leetcode.cn/problems/single-number-iii/)，要求：时间复杂度$O(n)$，空间复杂度$O(1)$
  - 提取一个数二进制位中最右侧的 1(其他位都置为 0): `a & (~a + 1)`



### 1.3 排序算法

#### 1.3.1 选择排序

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

时间复杂度：$O(n^2)$

#### 1.3.2 冒泡排序

```python
def bubble_sort(arr: List[int]):
    if not arr or len(arr) <= 1:
        return

    for i in range(len(arr) - 1, -1, -1):
        for j in range(i):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]

```

时间复杂度：$O(n^2)$

#### 1.3.3 插入排序

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

时间复杂度：$O(n^2)$；最好的情况（数组已经有序）下，时间复杂度$O(n)$

### 1.4 二分查找

#### 1.4.1 有序数组中，找某个数是否存在

时间复杂度：$O(log_2 N)$、$O(logN)$


#### 1.4.2 有序数组中，找 >= 某个数最左侧的位置

找到后还得继续往左找，一个变量记录上次位置，直到找到底

#### 1.4.3 无序数组中，局部最小值

[162. 寻找峰值](https://leetcode.cn/problems/find-peak-element/)

1. 假如索引 0，1 呈下降趋势，索引 n-1，n-2 也呈下降趋势，那么 0 和 n-1 中间必定有局部最小值
2. 中点索引 mid，如果 mid 值小于左边和右边，找到了，返回，否则
3. 假如 mid，mid-1 呈下降趋势，那么 0 和 mid 中间必定有局部最小值；右边同理
4. 二分继续

### 1.5 对数器的概念和使用

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

### 1.6 递归行为及其时间复杂度估计

master 公式，如果调用子问题规模相等，可以用该公式求时间复杂度：

$$T(N) = a * T(N / b) + O(N ^ d)$$

- $T(N)$：母问题的数据量为 $N$
- $T(N / b)$：子问题的规模是 $N / b$
- $a$：子问题调用次数
- $O(N ^ d)$：除了子问题调用，剩下的过程的时间复杂度

求解时间复杂度：
1. 如果 $log_b a > d$: $O(N ^ (log_b a))$
2. 如果 $log_b a == d$: $O((N ^ d) * log N)$
3. 如果 $log_b a < d$: $O(N ^ d)$

## 2.O(NlogN) 的排序

### 2.1 归并排序

代码：

```python
def merge_sort(arr: List[int]) -> None:
    if not arr or len(arr) <= 1:
        return

    process(arr, 0, len(arr) - 1)


def process(arr: List[int], left: int, right: int) -> None:
    if left == right:
        return

    mid = left + (right - left) // 2
    process(arr, left, mid)
    process(arr, mid + 1, right)
    merge(arr, left, mid, right)


def merge(arr: List[int], left: int, mid: int, right: int) -> None:
    tmp = []
    p = left
    q = mid + 1
    while p <= mid and q <= right:
        if arr[p] < arr[q]:
            tmp.append(arr[p])
            p += 1
        else:
            tmp.append(arr[q])
            q += 1

    tmp.extend(arr[p:mid + 1])
    tmp.extend(arr[q:right + 1])

    arr[left:right + 1] = tmp
```

- 时间复杂度(可由 master 公式求解)：$O(NlogN)$
- 空间复杂度：$O(N)$

算法题：
- [计算数组的小和](https://www.nowcoder.com/practice/6dca0ebd48f94b4296fc11949e3a91b8)
- [剑指 Offer 51. 数组中的逆序对](https://leetcode.cn/problems/shu-zu-zhong-de-ni-xu-dui-lcof/)
- [315. 计算右侧小于当前元素的个数](https://leetcode.cn/problems/count-of-smaller-numbers-after-self/)


### 2.2 快速排序

提出问题：

1. 问题一，给定一个数组 arr， 和一个数 num，请把小于等于 num 的数放在数组左边，大于 num 的数放在右边。要求额外空间复杂度 $O(1)$，时间复杂度 $O(N)$。
2. 问题二，给定一个数组 arr， 和一个数 num，请把小于 num 的数放在数组左边，等于 num 的数放在数组的中间，大于 num 的数放在右边。要求额外空间复杂度 $O(1)$，时间复杂度 $O(N)$

解决方案：

1. 问题一，双指针：一个遍历数组，一个指向小于等于 num 的子序列右边
2. 问题二，三指针：一个遍历数组，一个指向小于 num 的子序列右边，一个指向大于 num 子序列左边

引申快速排序：

1. 问题一：快速排序 1.0，在问题一基础上做递归，最后一个数做 num，问题一完成后，把最后一个数换到大于 num 子序列前面，即一次递归排好一个数
2. 问题二：快速排序 2.0，在问题二基础上做递归，最后一个数做 num，问题二完成后，把最后一个数换到大于 num 子序列前面，小于 num 子序列和大于 num 子序列指针中间的区域就有一批数排好了，即一次递归排好一批数

时间复杂度：

1. 快速排序 1.0：$O(N ^ 2)$。最差情况：数组已经有序，每次只有左序列，没有右序列，遍历次数为：n -1 + n - 2 + n - 3 ... = $O(N ^ 2)$
2. 快速排序 2.0：$O(N ^ 2)$。最差情况：数组已经有序，且不重复，每次只有左序列，没有右序列，遍历次数为：n -1 + n - 2 + n - 3 ... = $O(N ^ 2)$

快速排序 3.0：

- 由于划分值(1.0 和 2.0 都选当前总序列最后一个数)可能很偏，导致时间复杂度上升
- 如果划分值很打到中间位置，那么左右两个子问题的规模是一样的，由 `master 公式` 求解，时间复杂度将是 $O(NlogN)$
- 快速排序 3.0：随机选一个数，放到最后作为划分值，时间复杂度变成概论事件，求所有情况的数学期望，得时间复杂度为 $O(NlogN)$
- 时间复杂度 $O(NlogN)$，最差情况(每次都选到最差的那个)下 $O(N ^ 2)$
- 空间复杂度：$O(logN)$(递归栈类似二叉树)，最差情况(每次都选到最差的那个)下 $O(N)$



快速排序 3.0 代码：

```python
def quick_sort(arr: List[int]) -> None:
    if not arr or len(arr) <= 1:
        return

    process(arr, 0, len(arr) - 1)


def process(arr: List[int], left: int, right: int) -> None:
    if left < right:
        random_ix = random.randint(left, right)
        arr[random_ix], arr[right] = arr[right], arr[random_ix]  # 随机选一个数交换最后一个数，作为 partition 的基准

        left_end, right_start = partition(arr, left, right)
        process(arr, left, left_end)
        process(arr, right_start, right)


def partition(arr: List[int], left: int, right: int) -> tuple[int, int]:
    lt_end_next = left  # 左区域的右边一个位置
    gt_start_prev = right - 1  # 右区域的左边一个位置
    i = left
    while i <= gt_start_prev:
        if arr[i] < arr[right]:
            arr[lt_end_next], arr[i] = arr[i], arr[lt_end_next]
            lt_end_next += 1
            i += 1
        elif arr[i] > arr[right]:
            arr[gt_start_prev], arr[i] = arr[i], arr[gt_start_prev]
            gt_start_prev -= 1
        else:
            i += 1

    arr[gt_start_prev + 1], arr[right] = arr[right], arr[gt_start_prev + 1]
    gt_start_prev += 1

    return lt_end_next - 1, gt_start_prev + 1

```

算法题：
- [75. 颜色分类](https://leetcode.cn/problems/sort-colors/)

## 3. 堆排序和桶排序

### 3.1 堆和堆排序

#### 3.1.1 堆(Heap)

基本概念：
- 满二叉树：是一个绝对的三角形，也就是说它的最后一层全部是叶子节点
- 完全二叉树：他的最后一层可能不是完整的，但绝对是右方的连续部分缺失，左边一定是连续存在的
- 完全二叉树可由数组描述，数组中 i 位置节点的相关节点：
    - 左子节点：2 * i + 1
    - 右字节点：2 * i + 2
    - 父节点：(i - 1) // 2
- 堆是一种特殊的完全二叉树，通常是一个可以被看做一棵完全二叉树的数组对象：
    - 最大堆：父节点的值大于等于每一个子节点的值
    - 最小堆：父节点的值小于等于每一个子节点的值

python 实现：

```python
from numbers import Real


class MaxHeap:

    def __init__(self, datas: list[Real] | None = None):
        self._heap = []  # 维护堆的数组
        self._heap_size = 0  # 堆的大小
        if datas:
            self.build(datas)

    def insert(self, value: Real) -> None:
        """ 插入 """
        self._heap.append(value)
        self._heap_size += 1
        self._shift_up(self._heap_size - 1)

    def pop(self) -> Real | None:
        """ 返回并移除堆顶元素 """
        if self.is_empty():
            return None
        rtn = self._heap[0]
        self._heap[0] = self._heap[self._heap_size - 1]
        self._heap_size -= 1
        self._shift_down(0)
        return rtn

    def remove(self, index: int) -> Real:
        """ 返回并移除指定元素 """
        if index < 0 or index >= self._heap_size:
            raise IndexError

        rtn = self._heap[index]
        self._heap[index] = self._heap[self._heap_size - 1]
        self._heap_size -= 1
        modified = self._shift_up(index)
        if not modified:
            self._shift_down(index)
        return rtn

    def replace(self, value: Real, index: int = 0) -> None:
        """ 返回并替换指定元素元素 """
        if index < 0 or index >= self._heap_size:
            raise IndexError

        rtn = self._heap[index]
        self._heap[index] = value
        modified = self._shift_down(index)
        if not modified:
            self._shift_up(index)

        return rtn

    def is_empty(self) -> bool:
        """ 检查是否为空 """
        return self._heap_size == 0

    def peek(self):
        """ 返回堆顶元素 """
        if self.is_empty():
            return None
        return self._heap[0]

    def all(self) -> list[Real]:
        """ 返回所有元素 """
        return self._heap[:self._heap_size]

    def build(self, datas: list[Real]):
        """ 由数组创建堆，覆盖当前的堆 """
        self._heap = datas
        self._heap_size = len(datas)
        for i in range(self._heap_size - 1, -1, -1):
            self._shift_down(i)

    def _shift_up(self, index: int) -> bool:
        """
        向上调整堆
        时间复杂度：O(logN)
        :param index:
        :return: 是否修改了堆
        """
        if index < 0 or index >= self._heap_size:
            raise IndexError

        modified = False
        parent = max((index - 1) // 2, 0)
        while self._heap[index] > self._heap[parent]:
            modified = True
            self._heap[index], self._heap[parent] = self._heap[parent], self._heap[index]
            index = parent
            parent = max((index - 1) // 2, 0)

        return modified

    def _shift_down(self, index: int) -> bool:
        """
        向下调整堆
        时间复杂度：O(logN)
        :param index:
        :return: 是否修改了堆
        """
        if index < 0 or index >= self._heap_size:
            raise IndexError

        modified = False
        left = index * 2 + 1
        while left < self._heap_size:
            if left + 1 < self._heap_size and self._heap[left + 1] > self._heap[left]:
                largest = left + 1
            else:
                largest = left
            largest = largest if self._heap[largest] > self._heap[index] else index
            if largest == index:
                break

            self._heap[largest], self._heap[index] = self._heap[index], self._heap[largest]
            modified = True
            index = largest
            left = index * 2 + 1
        return modified
```

#### 3.1.2 堆排序

python 实现：

```python
def heap_sort(arr: list[int]) -> None:
    length = len(arr)
    if not arr or length <= 1:
        return

    # for i in range(length):
    #     shift_up(arr, i)

    for i in range(length - 1, -1, -1):  # 这种方法会使建堆过程复杂度由 O(NlogN) 降为 O(N)
        shift_down(arr, i, length)

    for i in range(length - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]
        shift_down(arr, 0, i)


def shift_down(arr: list[int], index: int, size: int) -> None:
    left = index * 2 + 1
    while left < size:
        if left + 1 < size and arr[left + 1] > arr[left]:
            largest = left + 1
        else:
            largest = left

        if arr[largest] > arr[index]:
            arr[index], arr[largest] = arr[largest], arr[index]
            index = largest
            left = index * 2 + 1
        else:
            break


# def shift_up(arr: list[int], index: int) -> None:
#     parent = max((index - 1) // 2, 0)
#     while arr[index] > arr[parent]:
#         arr[parent], arr[index] = arr[index], arr[parent]
#         index = parent
#         parent = max((index - 1) // 2, 0)
```

- 时间复杂度：$O(NlogN)$
- 空间复杂度：$O(1)$
- **优先级队列**结构就是**堆结构**

> 建堆的时候用 `shift_up` 时间复杂度为 $O(NlogN)$，而先把整个数组作为堆，再从最后位置开始 `shift_up`，时间复杂度可将为 $O(N)$(错位相减法求时间复杂度)

算法题：

- 已知一个几乎有序的数组，几乎有序是指，如果把数组排好顺序的话，每个元素移动的距离可以不超过k，  并且k相对于数组来说比较小。请选择一个合适的排序算法针对这个数据进行排序。

```python
import heapq


def solution(arr, k):
    length = len(arr)
    k = min(length, k)
    tmp = []
    ans = []
    i = 0
    while i < k:
        heapq.heappush(tmp, arr[i])
        i += 1

    while i < length:
        ans.append(heapq.heappop(tmp))
        heapq.heappush(tmp, arr[i])
        i += 1

    while tmp:
        ans.append(heapq.heappop(tmp))

    arr[:] = ans
```

### 3.2 桶排序

- 将数组分到有限数量的桶子里，每个桶子再分别别排序
- 桶排序不是比较排序：
    - 计数排序：员工年龄排序，准备 201 长度的数组，按下标计数每个年龄员工的个数，在遍历桶恢复原数组
    - **基数排序**：一种非比较型整数排序算法，其原理是将整数按位数切割成不同的数字，然后按每个位数分别比较

基数排序，python 实现，**仅适用于所有元素大于等于 0 的数组**：

```python
def max_bits(arr: list[int]) -> int:
    """ 获取数组中最大数的位数，如 max_bits([1,12,123,1234]) -> 4 """
    max_num = arr[0]
    for n in arr:
        if n > max_num:
            max_num = n

    res = 0
    while max_num != 0:
        max_num //= 10
        res += 1

    return res


def process(arr: list[int], digit: int) -> None:
    """
    做桶排序，只适用于元素大于等于 0 的数组
    :param arr:
    :param digit: 列表中最大数字的长度
    :return:
    """
    radix = 10  # 根据基数，确定 count 列表长度
    bucket = [0] * len(arr)  # 桶，不做实际的很多个桶，以 count 列表记录偏移量，从右到左遍历 arr 即可完成如桶和出桶操作
    for d in range(1, digit + 1):  # 最大几位数，就做几次入桶出桶
        count = [0] * radix  # 初始化 count 列表

        for n in arr:  # 记录当前位上，个数字()的出现次数
            nd = get_digit_num(n, d)
            count[nd] += 1

        for i in range(1, radix):  # 把出现次数，转化位偏移量，即小于等于 i 的有多少个
            count[i] += count[i - 1]

        for i in range(len(arr) - 1, -1, -1):  # 根据偏移量，把 arr 里的元素存入桶里，同时相当于做了出桶操作（由于桶是按照当前位的大小排序好的，小的在前面，大的在后面）
            nd = get_digit_num(arr[i], d)
            count[nd] -= 1
            bucket[count[nd]] = arr[i]

        arr[:] = bucket  # 桶覆盖原数组，相当于实际出桶


def get_digit_num(num: int, digit: int):
    """
    获取对应位上的值， 如 get_digit_num(456, 2) -> 5
    :param num: 原始数字
    :param digit: 从右往左数第几位
    :return: 对应位上的值，0 - 9
    """
    return (num // 10 ** (digit - 1)) % 10
```

## 4. 排序算法总结

### 4.1 排序算法的稳定性

- 稳定的：
    - 冒泡排序：相等时不交换
    - 插入排序：相等时不交换
    - 归并排序：相等时先考虑左侧
    - 桶排序：先入桶的先出桶，是稳定的
- 不稳定的：
    - 选择排序：选择后，交换的时候破坏稳定性
    - 快速排序：partition 交换的时候破坏稳定性
    - 堆排序：很轻易就破坏稳定性，建堆的时候就破坏稳定性了

### 4.2 总结

| 排序算法 | 时间复杂度      | 空间复杂度   | 稳定性 |
|------|------------|---------|-----|
| 选择排序 | $O(N ^ 2)$ | O(1)    | F   |
| 冒泡排序 | $O(N ^ 2)$ | O(1)    | T   |
| 插入排序 | $O(N ^ 2)$ | O(1)    | T   |
| 归并排序 | $O(NlogN)$ | O(N)    | T   |
| 快速排序 | $O(NlogN)$ | O(logN) | F   |
| 堆排序  | $O(NlogN)$ | O(1)    | F   |

- 归并排序是**稳定**的
- 归并排序和快速排序虽然都是 $O(NlogN)$，但通过实验，常数项更小从常数时间来讲**快速排序更快**，能用快排用快排
- **空间**十分紧张用堆排序
- 基于比较的排序，时间复杂度不低于$O(NlogN)$
- 时间复杂度 $O(NlogN)$ 的排序算法还要稳定，空间复杂度不低于 $O(N)$
- **为了绝对的速度选快排、为了省空间选堆排、为了稳定性选归并**

### 4.3 常见的坑

1. 归并排序的额外空间复杂度可以变成 $O(1)$，但非常难，感兴趣搜“归并排序内部缓存法”
2. “原地归并排序”都是垃圾，会让归并的时间复杂度变成 $O(N ^2)$
3. 快速排序可以做到稳定性，但会让空间复杂度变为 $O(N)$，但非常难，搜“01 stable sort”
4. 所有的改进都不重要，目前没有找到时间复杂度$O(NlogN)$，额外空间复杂度$O(1)$，又稳定的排序算法
5. 有一道题目，奇数放在数组左边，偶数放在数组右边，还要求原始的相对次序不变，时间复杂度 $O(N)$，额外空间复杂度 $O(1)$，碰到这个问题，可以怼面试官
    - 和快排类型，是一种 01 问题，和快排类似，如果 partition 能做到就可以，但 partition 交换会打乱次序
    - 能做到，但很难，搜 “01 stable sort”

### 4.4 工程上对排序的改进

1. 充分利用 $O(NlogN)$ 和 $O(N ^ 2)$ 各自的优势
    - 如快排实际应用，在递归到小样本量时（如 `length <= 60`），用插入排序，因为插入排序常数项小，且在数组几乎有序时效率高，这种情况下，比继续 partition 更有优势
2. 稳定性考虑


---

> 作者: [黄波](https://boh5.github.io)  
> URL: https://boh5.github.io/posts/notes/datastructure_algorithm/zuochengyun/1-sort-algorithms/  

