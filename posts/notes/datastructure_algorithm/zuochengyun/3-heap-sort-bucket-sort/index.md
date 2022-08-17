# 数据结构和算法学习笔记（三）堆排序和桶排序


## 1. 堆和堆排序

### 1.1 堆(Heap)

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

### 1.2 堆排序

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

- 时间复杂度：O(NlogN)
- 空间复杂度：O(1)
- **优先级队列**结构就是**堆结构**

> 建堆的时候用 `shift_up` 时间复杂度为 O(NlogN)，而先把整个数组作为堆，再从最后位置开始 `shift_up`，时间复杂度可将为 O(N)(错位相减法求时间复杂度)

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

## 2. 桶排序

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


---

> 作者: [黄波](https://dilless.github.io)  
> URL: https://dilless.github.io/posts/notes/datastructure_algorithm/zuochengyun/3-heap-sort-bucket-sort/  

