---
title: 'Python|寻找最大最小的N个元素几种方法'
date: 2017-09-15 09:48:44
tags:
  - Python
  - sort
categories: Python
---

实际的生产中，常常会需要处理一个序列，找出其中的 N 个最大或者最小的元素，这里提供几种思路，不同的情况，使用不同的搜索方式，可以更好提高我们代码的运行效率。

<!--more-->

###### “这里先假设目标序列的元素总数为 S ”

## N 如果是 1

如果只是简单地想找到最小或最大的元素，那么`min()`和`max()`是最合适的选择。

```Python
	min([1,2,3]) #1
	max([1,2,3]) #3
```

## N 如果想对 S 比较小

这种情况最高效的方式是使用函数`nlargest()`和`nsmallest()`

```Python
	import heapq
	heapq.nlargest(N, nums) #最大的N个元素组成的列表
	heapq.nsmallest(N, nums) #最小的N个元素
```

对于 N 较小的情况，还有一种高效的方式是下面这样：

heapq 的方式，这种方式首先会在底层将数据转为成列表，且元素会以堆得顺序排列。

```Python
	nums = [2,4,52,6,90,1]
	import heapq
	heap = list(nums)
	heapq.heapify(heap)
	heap
	[out]:[1, 4, 2, 6, 90, 52]
```

堆最重要的特性就是 `heap[0]`永远是最小的那个元素。接下来的元素，可以依次通过`heapq.heappop()`方法找到，这个方法会将第一个元素，也就是最小的元素弹出，然后以第二小的元素代替。这种操作的复杂度为`O(logN)`

```Python
	heapq.heappop(heap)
	[out]:1
	heapq.heappop(heap)
	[out]:2
```

## 如果 N 和 S 相近

这种情况下，通常更快的方法是先对集合排序，然后做切片操作。

```Python
	sorted(nums)[:N]
	#或者
	sorted(nums)[-N:]
```

---

> - 版权声明：欢迎转载，请注明出处
> - 发表日期：2017-09-15
> - 博客地址：blog.vimiix.com
