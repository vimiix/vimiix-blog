---
title: '《算法图解》读书笔记3-递归'
date: 2018-05-25 19:20:48
categories: 'note'
tags: ['algorithms', 'Linux']
---

> 如果使用循环 ，程序的性能可能更高；如果使用递归，程序可能更容易理解。如何选择要看什么对你来说更重要。

## 递归函数

在一个函数中，可以调用另一个函数，如果调用的另一个函数是函数本身，这样的函数就是递归函数。如下示例：

```python
def foo(x):
    if x == 0:
        return 'end'
    else:
        # 在函数中继续调用自己
        return foo(x-1)
```

<!--more-->

递归函数中，必须有两个条件，来保证函数的能够有效运行并返回。

- 基线条件(base case)：即函数停止调用自己的条件，避免无限循环调用产生[栈溢出](https://zh.wikipedia.org/wiki/%E5%A0%86%E7%96%8A%E6%BA%A2%E4%BD%8D)，导致程序崩溃；
- 递归条件(reversive case)：即函数调用自己的条件，由此形成递归；

还以上面的代码实例说明：`x == 0`就是该递归函数的基线条件，`else` 分支即为递归条件

## 栈

**栈**是一种数据结构，它的结构类似一个桶，只有一个开口处可以进出元素。每次从桶的开口处，放入一个元素，这种操作叫做**“压栈”**（push），一层一层的摞起来，每次读取或删除的时候，只能从最顶部开始一个一个取出，这个操作叫做**“出栈”**（pop）。桶的底部称为**“栈底”**（bottom），桶内最高处的元素（也就是最后放进去的元素）所在的内存地址称为**栈顶**（top）,通过下面一个小动图展示栈的工作原理：

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/stack.gif)

使用 python 内建数据类型`list`实现栈的数据结构：

```python
class Stack:
    def __init__(self):
        # 造一个桶
        self.items = []

    # 压栈
    def push(self, item):
        self.items.append(item)

    # 出栈
    def pop(self):
        return self.items.pop()

    # 清空栈
    def clear(self):
        del self.items[:]

    # 当前栈的大小(元素个数)
    def size(self):
        return len(self.items)

    # 栈是否为空
    def empty(self):
        return self.size() == 0

    # 栈顶元素
    def top(self):
        return self.items[self.size()-1]
```

## 调用栈

（引自[维基百科](https://zh.wikipedia.org/wiki/%E5%91%BC%E5%8F%AB%E5%A0%86%E7%96%8A)）

计算机中使用调用栈来存放子程序的返回地址，即当子程序运行结束后应该返回的地址。如果被调用的子程序还要调用其他的子程序，其自身的返回地址就继续压栈到调用栈，在其自身运行完毕后再自行取回。在递归程序中，每一层次递归都必须在调用栈上压栈一条地址，因此如果程序出现无限递归（或仅仅是过多的递归层次），调用栈就会产生栈溢出。（优化方式：**尾递归**优化，实现递归空间复杂度 O(1)）

**调用栈的功能**:

调用栈的主要功能是存放[返回地址](https://zh.wikipedia.org/w/index.php?title=%E8%BF%94%E5%9B%9E%E4%BD%8D%E5%9D%80&action=edit&redlink=1)。除此之外，调用栈还用于存放：

- [本地变量](https://zh.wikipedia.org/wiki/%E6%9C%AC%E5%9C%B0%E8%AE%8A%E6%95%B8)：子程序的变量可以存入调用栈，这样可以达到不同子程序间变量分离开的作用。
- [参数](https://zh.wikipedia.org/wiki/%E5%8F%82%E6%95%B0)传递：如果[寄存器](https://zh.wikipedia.org/wiki/%E6%9A%AB%E5%AD%98%E5%99%A8)不足以容纳子程序的参数，可以在调用栈上存入参数。
- 环境传递：有些语言（如[Pascal](https://zh.wikipedia.org/wiki/Pascal)与[Ada](https://zh.wikipedia.org/wiki/Ada)）支持“多层子程序”，即子程序中可以利用主程序的本地变量。这些变量可以通过调用栈传入子程序。

## 递归应用举例：汉诺塔游戏

学习递归，最经典的一个例子就是通过使用递归实现汉诺塔的解决方案。通俗的代码逻辑，把中间过程的一切逻辑过程都交给计算机去处理。

**游戏规则**：

有三根杆子 A，B，C。A 杆上有 N 个(N>1)穿孔圆盘，盘的尺寸由下到上依次变小。要求按下列规则将所有圆盘移至 C 杆：

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/hanoi.jpg)

1. 每次只能移动一个圆盘；
2. 大盘不能叠在小盘上面。

提示：可将圆盘临时置于 B 杆，也可将从 A 杆移出的圆盘重新移回 A 杆，但都必须遵循上述两条规则。

问：如何移？最少要移动多少次？

借助递归，可以清晰简洁的实现解法：

```python
def move(n, A, B, C):
    # 基线条件
    if n == 1:
        print(A, "->", C)
        return
    # 将最底下一层上面的所有视为一个整体，借助C移动到B
    move(n-1, A, C, B)
    # 将最底下的一层，借助B移动到C
    move(1, A, B, C)
    # 将B上剩下的所有层，借助A移动到C
    move(n-1, B, A, C)

# 测试三层移动结果
>>> move(3, 'A', 'B', 'C')
A -> C
A -> B
C -> B
A -> C
B -> A
B -> C
A -> C
>>>
```

— EOF —
