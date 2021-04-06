---
title: '[算法笔记]用递归和迭代的思想分别实现插入排序和选择排序'
date: 2017-08-02 17:17:25
tags:
  - python
  - algorithms
categories: 'note'
---

# 思路

<hr>

### 插入排序（insert sort）

> 先归纳性的假设前 n-1 个元素已经完成排序了，现在要将第 n 个元素<font color='red'>向前插入</font>到正确的位置上。这种方式称之为**插入排序法**

### 选择排序（selection sort）

> 先找到整个序列中最大的元素，并将其<font color='red'>向后放</font>在 n（待排序的序列末尾）的位置上，然后继续递归排序剩下的元素。这种方式称之为**选择排序法**

<!--more-->

# Python 代码实现

<hr>

### 递归版插入排序

```Python

	def ins_sort_rec(seq, i):
	  if i == 0:                                  #单元素情况，直接返回
	    return
	  ins_sort_rec(seq, i-1)                      #递归排序0~(i-1)的元素，排好后，后面会将i的元素向前插入
	  j = i
	  while j > 0 and seq[j-1] > seq[j]:          #寻找合适的位置，直到找到比seq[j]小的元素时，停止循环
	    seq[j-1], seq[j] = seq[j], seq[j-1]       #交换位置
	    j -= 1                                    #递减索引值
```

这段代码概括了该算法的思路：如果想要对目标序列中的第 i 个元素进行排序，首先要对 i-1 个元素进行递归性排序，并通过交换方式将`seq[i]`放到其已排序元素中的正确位置上，。尽管这种实现形式可以让我们在递归调用中很好的概括其归纳前提，但在具体实践中会受到限制（_例如：它只能在一定的序列长度下正常工作_）。

### 迭代版插入排序

```Python

	def ins_sort(seq):
	  for i in range(1,len(seq)):                  #从0到(i-1)排序
	    j = i                                      #j作为扫描索引从后向前扫描，i为每一次的要插入的序列长度
	    while j > 0 and seq[j-1] > seq[j]:         #寻找合适位置
	      seq[j-1], seq[j] = seq[j], seq[j-1]      #依次交换位置
	      j -= 1                                   #递减索引值扫描
```

上面这段代码所展示的则是更为人们所熟知的迭代版插入排序。它将原先的后退式递归调用改成了从第一个元素开始的前进式迭代操作。但仔细推敲一下，就会发现递归也是这样做的。尽管看起来递归似乎是从序列尾端开始的，但是 while 循环被执行之前，这些递归调用得要先完全回退到第一个元素上。然后在该递归调用开始返回之后，while 循环才能开始处理第二个元素，并以此类推下去。所以，以上两个版本的行为其实是相同的。

### 递归版选择排序

```Python

	def sel_sort_rec(seq, i):
	  if i == 0:                                   #单元素直接返回
	    return
	  max_j = i                                    #初始化一个max_j作为要找的最大元素的索引值，先假设目前序列的最后一个为最大值
	  for j in range(i):                           #寻找最大元素
	    if seq[j] > seq[max_j]:
	      max_j = j                                #找到一个比seq[i]大的就更新max_j的值
	  seq[i], seq[max_j] = seq[max_j], seq[i]      #将最大的元素放到末尾
	  sel_sort_rec(seq, i-1)                       #继续排列前i-1个元素
```

### 迭代版选择排序

```Python

	def sel_sort(seq):
	  for i in range(len(seq)-1, 0, -1):           #从最长开始依次递减将最大元素放在末尾
	    max_j = i                                  #初始化max_j的索引值
	    for j in range(i):                         #寻找最大值
		  if seq[j] > seq[max_j]:
	        max_j = j                              #更新最大元素的索引值
	    seq[i], seq[max_j] = seq[max_j], seq[i]    #奖最大元素后放
```

这两段代码和上面一样，递归版实现明确表示了其归纳前提，而迭代版则明确说明了其反复迭代执行的归纳步骤。两者都是先找出最大元素，并将其交换到当前所关注序列的尾端。
