---
title: '《算法图解》读书笔记2-数组链表和选择排序'
date: 2018-05-25 19:20:48
categories: 'note'
tags: ['algorithms', 'linux', 'sort']
---

## 理解数组和链表

链表和数组是两种基本的数据结构，他们的区别在于数据在内存中的存储方式不同。

### 数组

数组在内存中是用一块连续的内存来存储数据的，数组中的每个数据地址是连续的。数组中的每个元素所占用的内存是相同的，所以，我们可以通过下标索引在常数数量级的时间内，迅速访问数组中的任何一个元素。但是要在数组中任意位置添加一个元素，就需要移动大量的元素，使得内存中空出一个位置来存放新插入的元素。同理，当删除一个元素的时候，也需要移动大量的元素，来使得删除元素以后的数组数据在内存中仍旧是连续的。

由此可见：当对于一组数据，读取操作频繁，写操作少的情况，应该使用数组数据结构。

<!--more-->

### 链表

链表与数组相反，链表中的元素在内存中是随机存放的，链表中的每个元素都存放了当前元素的值以及下一个数据的内存地址指针。通过这个地址指针，使得随机存放的数据，得以相互之间连接起来，形成链表。如果我们需要访问链表中的某个元素的时候，需要从链表的第一个元素开始逐个遍历寻找，直到找到目标元素。这一点来说，远远不如数组的访问效率高。但是链表的优势在于，当我们想要插入或删除一个元素的时候，只需要处理一下要插入或删除的元素前后元素的地址指针，就可以完成。修改之后，不需要移动其他数据。

由此可见：当对于一组数据，增加删除元素操作频繁，读取操作少的时候，应该使用链表数据结构

### 二者区别

1. 数组数据是连续的，一般需要预先设定数据长度，不能适应数据动态的增减，当数据增加是可能超过预设值，需要要重新分配内存，当数据减少时，预先申请的内存未使用，造成内存浪费。链表的数据因为是随机存储的，所以链表可以动态的分配内存，适应长度的动态变化；
2. 数据的元素是存放在栈中的，链表的元素在堆中；
3. 读取操作：数组时间复杂度为 O(1)，链表为 O(n)
4. 插入或删除操作：数据时间复杂度为 O(n)，链表为 O(1)

## 选择排序

### 选择排序原理

> 每一轮比较找到一个极值（最大值或最小值）放到某一端，对剩下的数再找极值，直至比较结束。

开始学习算法的时候，对选择排序和冒泡有点混淆，这两种排序算法都是从待排序的列表中寻找最大或最小值，然后移动到最旁边。但这两种算法有些区别

- 冒泡排序的思想是：每一次排序过程，通过相邻元素的交换，将当前没有排好序中的最大（小）移到数组的最右（左）端。
- 选择排序的思想是：每一次排序过程，我们获取当前没有排好序中的最大（小）的元素和数组最右（左）端的元素交换，循环这个过程即可实现对整个数组排序。

### 选择排序的动画演示

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/SelectionSort_Avg_case.gif)

### 基本实现

```python
def selection_sort(arr):
    length = len(arr)
    if length < 2:
        return arr
    for out_idx, base in enumerate(arr):
        min = out_idx
        # 遍历剩下的数据
        for inner_idx in range(out_idx+1, length):
            if arr[inner_idx] < base:
                min = inner_idx # 记录最小值的索引
        if out_idx != min:
            # 交换当前位置的数据和最小索引处的值
            arr[out_idx], arr[min] = arr[min], arr[out_idx]
    return arr
```

### 复杂度

| 维度           | 复杂度  |
| -------------- | ------- |
| 最坏时间复杂度 | О(n²)   |
| 最优时间复杂度 | _О(n²)_ |
| 平均时间复杂度 | _О(n²)_ |

### 优化实现：二元选择排序

每次确定两个数（最大值和最小值），减少迭代次数

```python
def selection_sort(arr):
    length = len(arr)
    if length < 2:
        return arr
    for i in range(length//2):
        min = i
        max = -i - 1
        max_origin = max

        # 左右两边同时交叉遍历
        for j in range(i+1, length-i):
            if arr[j] < arr[min]:
                min = j
            if arr[-j - 1] > arr[max]:
                max = -j - 1

        if i != min:
            arr[i], arr[min] = arr[min], arr[i]
            # 如果此时恰好max也指向i，i被移动了，所以max也要相应的移动
            if i == max or i == length + max:
                max = min
        if max_origin != max:
            arr[max_origin], arr[max] = arr[max], arr[max_origin]
    return arr

```

### 优化实现：等值情况

二元选择排序的时候，每一轮可以知道最大值和最小值，如果某一轮最大最小值都一样了，说明剩下的数字都是相等的，直接结束排序。

```python
def selection_sort(arr):
    length = len(arr)
    if length < 2:
        return arr
    for i in range(length//2):
        min = i
        max = -i - 1
        max_origin = max

        for j in range(i+1, length-i):
            if arr[j] < arr[min]:
                min = j
            if arr[-j - 1] > arr[max]:
                max = -j - 1

        # 如果最大最小值相等
        if arr[min] == arr[max]:
            break

        if i != min:
            arr[i], arr[min] = arr[min], arr[i]
            if i == max or i == length + max:
                max = min
        if max_origin != max:
            arr[max_origin], arr[max] = arr[max], arr[max_origin]
    return arr
```

### 优化实现：等值情况优化

如果`arr` = `[2,1,1,1,1]`,找到最大值索引为-5, 最小值索引为 1，上面代码会交换两次，第二次的两个 1 交换是多余的操作，所以添加一个判断，如果值相同则不交换。

```python
def selection_sort(arr):
    length = len(arr)
    if length < 2:
        return arr
    for i in range(length//2):
        min = i
        max = -i - 1
        max_origin = max

        for j in range(i+1, length-i):
            if arr[j] < arr[min]:
                min = j
            if arr[-j - 1] > arr[max]:
                max = -j - 1
        if arr[min] == arr[max]:
            break

        if i != min:
            arr[i], arr[min] = arr[min], arr[i]
            if i == max or i == length + max:
                max = min
        # 添加判断，如果两个索引处的数据相同，不交换
        if max_origin != max and arr[max_origin] != arr[max]:
            arr[max_origin], arr[max] = arr[max], arr[max_origin]
    return arr
```

—EOF—
