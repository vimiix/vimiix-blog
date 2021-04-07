---
title: '《算法图解》读书笔记1-二分和大O'
date: 2018-05-23 23:20:48
categories: 'note'
tags: ['algorithms', 'Linux']
---

> 算法是一组完成任务的指令

## 为什么要学习算法

这次是我第二次读《算法图解》，当我第一次看这本书的时候，我更兴奋于书中有什么内容，迫不及待的去过内容，学习那些算法概念。但当我第二次准备开始读这本书的时候，我脑海中出现的了一个问题：“为什么要学习算法？”，这个问题也许会有人和我一样，之前根本没有好好的去思考，只是知道，作为一个程序员我应该学习算法。当然，能够有这个觉悟，说明我们还算是个合格的程序员。

但是，不妨认真思考一下，为什么要学习算法？算法应该怎么学？

<!--more-->

首先，我觉得算法是一种思想，并不只是一段代码。它不拘泥于形式，而在于我们去如何运用这种思想。我们目前接触到的算法，都可以算作是前辈们通过不断的实践经验而得出的一种总结。算法可以帮助我们写出性价比更高的程序，来应对实际生产中的种种挑战。既然算法是一种思想，那么他就不会是一陈不变的，在学习的过程中，要保持一颗怀疑的态度，不断的试图对已有算法举反例验证。若能针对已有的算法提出自己的改进，那真是一件有意义的事情。

再者，算法好比我们学语言的时候，需要先学习语法，才能够将学到的词汇灵活的整理成话语。说话的方式千奇百怪，不同的场合下，我们会选择说合适的话。算法也一样，不能单纯的从运行时间上对比两个算法的优劣，有的场合内存要求高，时间要求不高，这时候可以能就需要牺牲时间来满足内存要求。将不同的算法，放置到不同的生产要求环境中，再选择适宜的算法，这是我接下来学习每个算法应该思考的出发点。

最后，学好算法，有助于提升我们的技术水平，找到更好的工作机会，赚更多的钱。最后走上人生巅峰，迎娶白富美。[自行脑补......]

## 二分查找

二分查找算法（binary search），也称为折半搜索（英语：half-interval search），对数搜索（英语：logarithmic search）。是一种在有序数组中查找某一特定元素的搜索算法。

算法前提：

- 有序数组

算法步骤图示：

![](http://ww1.sinaimg.cn/large/8d56d744ly1frkjc3ktw7j20am06swec.jpg)

算法 python3 代码实现：(针对元素互不相同的情况，若有连续相同的，见讨论区补充)

- 递归实现

```python
def binary_search(list, item):
    low = 0
    high = len(list) - 1

    mid = (low + high)//2
    if list[mid] > item:
        return binary_search(list[:mid], item)
    if list[mid] < item:
        return binary_search(list[mid:], item)
    return mid
```

- 循环实现

```python
def binary_search(list, item):
    low = 0
    high = len(list) - 1

    while low <= high:
        mid = (low + high)//2
        if list[mid] < item:
            low = mid + 1
        elif list[mid] > item:
            high = mid - 1
        else:
            return mid
```

算法复杂度：

- 最优时间复杂度：O(1)
- 最坏时间复杂度：O(logn)
- 平均时间复杂度：O(logn)
- 空间复杂度：迭代实现：O(1)；循环实现：O(logn)

## 大 O 表示法

大 O 表示法指出了算法的效率。

> Wikipedia：**大 O 符号**（英语：Big O notation），又称为**渐进符号**，是用于描述[函数](https://zh.wikipedia.org/wiki/%E5%87%BD%E6%95%B0)[渐近行为](https://zh.wikipedia.org/wiki/%E6%B8%90%E8%BF%91%E5%88%86%E6%9E%90)的[数学](https://zh.wikipedia.org/wiki/%E6%95%B0%E5%AD%A6)符号。更确切地说，它是用另一个（通常更简单的）函数来描述一个函数[数量级](https://zh.wikipedia.org/wiki/%E6%95%B0%E9%87%8F%E7%BA%A7)的**渐近上界**。

每个算法都可以通过一个数学公式表达，大 O 表示法，标记的是一个函数中起主导作用的一项，而其他项可以被忽略。举个例子，解决一个规模为 n 的问题所需步骤的数目可以表示为：

![T(n)=4n^{2}-2n+2](https://wikimedia.org/api/rest_v1/media/math/render/svg/679aea9b28e1092062bb5b79f3a89cde55e7710d)

当 n 增大时，n 平方项将主导整个算式结果的走向，其他项的影响将越来越小，甚至忽略不计。用大 O 表示法就是：

![T(n)=\mathrm {O} (n^{2})](https://wikimedia.org/api/rest_v1/media/math/render/svg/68b71c2ae4dc99e2e197d28c6537c475a1a07a73)

我们就说该算法具有平方阶的时间复杂度。

## 常用函数阶

![](http://ww1.sinaimg.cn/large/8d56d744ly1frkkjdko4vj20pk0j8dj1.jpg)
