# 数据结构和算法学习笔记（二）O(NlogN) 的排序


## 1. 归并排序

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

- 时间复杂度(可由 master 公式求解)：O(NlogN)
- 空间复杂度：O(N)

算法题：
- [计算数组的小和](https://www.nowcoder.com/practice/6dca0ebd48f94b4296fc11949e3a91b8)
- [剑指 Offer 51. 数组中的逆序对](https://leetcode.cn/problems/shu-zu-zhong-de-ni-xu-dui-lcof/)
- [315. 计算右侧小于当前元素的个数](https://leetcode.cn/problems/count-of-smaller-numbers-after-self/)


## 2. 快速排序

提出问题：

1. 问题一，给定一个数组 arr， 和一个数 num，请把小于等于 num 的数放在数组左边，大于 num 的数放在右边。要求额外空间复杂度 O(1)，时间复杂度 O(N)。
2. 问题二，给定一个数组 arr， 和一个数 num，请把小于 num 的数放在数组左边，等于 num 的数放在数组的中间，大于 num 的数放在右边。要求额外空间复杂度 O(1)，时间复杂度 O(N)

解决方案：

1. 问题一，双指针：一个遍历数组，一个指向小于等于 num 的子序列右边
2. 问题二，三指针：一个遍历数组，一个指向小于 num 的子序列右边，一个指向大于 num 子序列左边

引申快速排序：

1. 问题一：快速排序 1.0，在问题一基础上做递归，最后一个数做 num，问题一完成后，把最后一个数换到大于 num 子序列前面，即一次递归排好一个数
2. 问题二：快速排序 2.0，在问题二基础上做递归，最后一个数做 num，问题二完成后，把最后一个数换到大于 num 子序列前面，小于 num 子序列和大于 num 子序列指针中间的区域就有一批数排好了，即一次递归排好一批数

时间复杂度：

1. 快速排序 1.0：O(N ^ 2)。最差情况：数组已经有序，每次只有左序列，没有右序列，遍历次数为：n -1 + n - 2 + n - 3 ... = O(N ^ 2)
2. 快速排序 2.0：O(N ^ 2)。最差情况：数组已经有序，且不重复，每次只有左序列，没有右序列，遍历次数为：n -1 + n - 2 + n - 3 ... = O(N ^ 2)

快速排序 3.0：

- 由于划分值(1.0 和 2.0 都选当前总序列最后一个数)可能很偏，导致时间复杂度上升
- 如果划分值很打到中间位置，那么左右两个子问题的规模是一样的，由 `master 公式` 求解，时间复杂度将是 O(NlogN)
- 快速排序 3.0：随机选一个数，放到最后作为划分值，时间复杂度变成概论事件，求所有情况的数学期望，得时间复杂度为 O(NlogN)
- 时间复杂度 O(NlogN)，最差情况(每次都选到最差的那个)下 O(N ^ 2)
- 空间复杂度：O(logN)(递归栈类似二叉树)，最差情况(每次都选到最差的那个)下 O(N)



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


---

> 作者: [黄波](https://dilless.github.io)  
> URL: https://dilless.github.io/posts/notes/datastructure_algorithm/zuochengyun/2-nlogn-sort/  

