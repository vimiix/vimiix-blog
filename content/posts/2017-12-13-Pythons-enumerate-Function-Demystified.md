---
title: '[译]Python的enumerate()函数揭秘'
date: 2017-12-13 19:51:48
categories: 'translation'
tags: [python, 'translation', enumerate]
---

> 今天的原文的作者是来自国外的一位 Python“布道师”[Dan Bader](https://dbader.org)，他的博客完全就是一个个人品牌的学校。有跟多 Python 技巧，有很多他录制的 Youtube 视频，国内的 Pythonista 们，不妨订阅一下他的每周邮件推送。[订阅链接](https://www.getdrip.com/forms/80014959/submissions/new)
>
> 今天的译文是他博客中的一篇，点击[查看原文](https://dbader.org/blog/python-enumerate)

如何以去写以及为什么你应该使用 Python 中的内置枚举函数来编写更干净更加 Pythonic 的循环语句？

<!--more-->

![python-enumerate](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/python-enumerate.jpg)

Python 的`enumerate`函数是一个神话般的存在，以至于它很难用一句话去总结它的目的和用处。

但是，它是一个非常有用的函数，许多初学者，甚至中级 Pythonistas 是并没有真正意识到。简单来说，`enumerate()`是用来遍历一个可迭代容器中的元素，同时通过一个计数器变量记录当前元素所对应的索引值。

让我们来看一个示例：

```python
names = ['Bob', 'Alice', 'Guido']
for index, value in enumerate(names):
    print(f'{index}: {value}')
```

这段代码会输入如下内容：

```python
0: Bob
1: Alice
2: Guido
```

正如你所看到的，这个循环遍历了`names`列表的所有元素，并通过增加从零开始的计数器变量来为每个元素生成索引。

_[如果您想知道上面例子中使用的 f'...'字符串语法，这是 Python 3.6 及更高版本中提供的一种新的字符串格式化技巧。]_

## 用`enumerate()`让你的循环更加 Pythonic

那么为什么用`enumerate()`函数去保存运行中的索引很有用呢？

我发现，有很多从 C 或 Java 背景转过来的新的 Python 开发人员有时使用下面这种`range(len(...))`方法来保存运行中每个元素的索引，同时再用`for`循环遍历列表：

```python
# 警告: 不建议这么写
for i in range(len(my_items)):
    print(i, my_items[i])
```

通过巧妙地使用`enumerate()`函数，就像我在上面的“names”例子中写的那样，你可以[使你的循环结构看起来更 Pythonic](https://dbader.org/blog/pythonic-loops)和地道。

你不再需要在 Python 代码中专门去生成元素索引，而是将所有这些工作都交给`enumerate()`函数处理即可。这样，你的代码将更容易被阅读，而且减少写错代码的影响。_（译者注：写的代码越多，出错几率越高，尽量将自己的代码看起来简洁，易读，Pythonic，才是我们的追求）_

## 修改起始索引

另一个有用的特性是，`enumerate()`函数允许我们为循环自定义起始索引值。`enumerate()`函数中接受一个可选参数，该参数允许你为本次循环中的计数器变量设置初始值：

```python
names = ['Bob', 'Alice', 'Guido']
for index, value in enumerate(names, 1):
    print(f'{index}: {value}')
```

在上面的例子中，我将函数调用改为`enumerate(names, 1)`，后面的参数 1 就是本次循环的起始索引，替换默认的 0：

```python
1: Bob
2: Alice
3: Guido
```

OK，这段代码演示的就是如何将 Python 的`enumerate()`函数默认 0 起始索引值修改为 1（或者其他任何整形值，根据需求去设置不同值）

## `enumerate()`背后是如何工作的

你可能想知道`enumerate()`函数背后是如何工作的。事实上他的部分魔法是通过[Python 迭代器](https://dbader.org/blog/python-iterators)来实现的。意思就是每个元素的索引是懒加载的（一个接一个，用的时候生成），这使得内存使用量很低并且保持这个结构运行很快。

让我们演示一些更多的代码来表达我的意思：

```python
>>> names = ['Bob', 'Alice', 'Guido']
>>> enumerate(names)
<enumerate object at 0x1057f4120>
```

在上面这个代码片段中，正如你所见，我使用了和前面一样的示例代码。但是，调用`enumerate()`函数并不会立即返回循环的结果，而只是在控制台中返回了一个`enumerate`对象。

正如你所看到的，这是一个“枚举对象”。它的确是一个迭代器。就像我说的，它会在循环请求时懒加载地输出每个元素。

为了验证，我们可以取出那些“懒加载”的元素，我计划在这个迭代器上调用 Python 的内置函数`list()`

```python
>>> list(enumerate(names))
[(0, 'Bob'), (1, 'Alice'), (2, 'Guido')]
```

对于输入`list()`中的每个`enumerate()`迭代器元素，迭代器会返回一个形式为`（index，element）`的元组作为 list 的元素。在典型的 for-in 循环中，你可以利用 Python 的[数据结构解包](https://dbader.org/blog/python-nested-unpacking)功能来充分利用这一点特性：

```python
for index, element in enumerate(iterable):
    # ...
```

## 总结：Python 中的 enumerate 函数 - 关键点

- `enumerate`是 Python 的一个内置函数。你应该充分利用它通过循环迭代自动生成的索引变量。
- 索引值默认从 0 开始，但也可以将其设置为任何整数。
- `enumerate`函数是从 2.3 版本开始被添加到 Python 中的，详情见[PEP279](https://www.python.org/dev/peps/pep-0279/)。
- Python 的`enumerate`函数可以帮助你编写出更加 Pythonic 和地道的循环结构，避免使用笨重且容易出错的手动生成索引。
- 为了充分利用`enumerate`的特性，一定要研究 Python 的迭代器和[数据结构解包](https://dbader.org/blog/python-nested-unpacking)功能。
