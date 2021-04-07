---
title: '[译]关于python字典类型最疯狂的表达方式'
date: 2017-12-28 10:31:48
categories: 'Python'
tags: [Python, 'translation', dict, trick]
---

[![](https://badge.juejin.im/entry/5a4459c9f265da43112080ea/likes.svg?style=flat-square)](https://juejin.im/post/5a4459b05188257d6a7ed76b)

> 一篇来自 [Dan Bader](https://dbader.org/blog/python-mystery-dict-expression) 的有趣的博文，一起来学习一下，如何去研究一个意外的 Python 现象。

<!--more-->

## 一个 Python 字典表达式谜题

让我们探究一下下面这个晦涩的 python 字典表达式，以找出在 python 解释器的中未知的内部到底发生了什么。

```python
# 一个python谜题：这是一个秘密
# 这个表达式计算以后会得到什么结果？

>>> {True: 'yes', 1: 'no', 1.0: 'maybe'}
```

有时候你会碰到一个很有深度的代码示例 --- 哪怕仅仅是一行代码，但是如果你能够有足够的思考，它可以教会你很多关于编程语言的知识。这样一个代码片段，就像是一个*`Zen kōan`*：一个在修行的过程中用来质疑和考验学生进步的问题或陈述。

_译者注：`Zen kōan`,大概就是修行的一种方式，详情见[wikipedia](https://en.wikipedia.org/wiki/K%C5%8Dan)_

我们将在本教程中讨论的小代码片段就是这样一个例子。乍看之下，它可能看起来像一个简单的词典表达式，但是仔细考虑时，通过 cpython 解释器，它会带你进行一次思维拓展的训练。

我从这个短短的一行代码中得到了一个启发，而且有一次在我参加的一个 Python 会议上，我还把作为我演讲的内容，并以此开始演讲。这也激发了我的 python 邮件列表成员间进行了一些积极的交流。

所以不用多说，就是这个代码片。花点时间思考一下下面的字典表达式，以及它计算后将得到的内容：

```python
>>> {True: 'yes', 1: 'no', 1.0: 'maybe'}
```

在这里,我先等会儿，大家思考一下...

- 5...
- 4...
- 3...
- 2...
- 1...

---

OK, 好了吗？

这是在 cpython 解释器交互界面中计算上述字典表达式时得到的结果：

```python
>>> {True: 'yes', 1: 'no', 1.0: 'maybe'}
{True: 'maybe'}
```

我承认，当我第一次看到这个结果时，我很惊讶。但是当你逐步研究其中发生的过程时，这一切都是有道理的。所以，让我们思考一下为什么我们得到这个 - _我想说的是出乎意料_ - 的结果。

## 这个子字典是从哪里来的

当 python 处理我们的字典表达式时，它首先构造一个新的空字典对象;然后按照字典表达式给出的顺序赋键和值。

因此，当我们把它分解开的时候，我们的字典表达就相当于这个顺序的语句：

```python
>>> xs = dict()
>>> xs[True] = 'yes'
>>> xs[1] = 'no'
>>> xs[1.0] = 'maybe'
```

奇怪的是，Python 认为在这个例子中使用的所有字典键是相等的：

```python
>>> True == 1 == 1.0
True
```

OK，但在这里等一下。我确定你能够接受 1.0 == 1，但实际情况是为什么`True`也会被认为等于 1 呢？我第一次看到这个字典表达式真的让我难住了。

在 python 文档中进行一些探索之后，我发现 python 将`bool`作为了`int`类型的一个子类。这是在 Python 2 和 Python 3 的片段：

> “The Boolean type is a subtype of the integer type, and Boolean values behave like the values 0 and 1, respectively, in almost all contexts, the exception being that when converted to a string, the strings ‘False’ or ‘True’ are returned, respectively.”
>
> “布尔类型是整数类型的一个子类型，在几乎所有的上下文环境中布尔值的行为类似于值 0 和 1，例外的是当转换为字符串时，会分别将字符串”False“或”True“返回。“（[原文](https://docs.python.org/3/reference/datamodel.html#the-standard-type-hierarchy)）

是的，这意味着你可以在编程时上使用`bool`值作为 Python 中的列表或元组的索引：

```python
>>> ['no', 'yes'][True]
'yes'
```

但为了代码的可读性起见，您不应该类似这样的来使用布尔变量。（也请建议你的同事别这样做）

Anyway，让我们回过来看我们的字典表达式。

就 python 而言，`True`，`1`和`1.0`都表示相同的字典键。当解释器计算字典表达式时，它会重复覆盖键`True`的值。这就解释了为什么最终产生的字典只包含一个键。

在我们继续之前，让我们再回顾一下原始字典表达式：

```python
>>> {True: 'yes', 1: 'no', 1.0: 'maybe'}
{True: 'maybe'}
```

这里为什么最终得到的结果是以`True`作为键呢？由于重复的赋值，最后不应该是把键也改为`1.0`了？经过对 cpython 解释器源代码的一些模式研究，我知道了，当一个新的值与字典的键关联的时候，python 的字典不会更新键对象本身：

```python
>>> ys = {1.0: 'no'}
>>> ys[True] = 'yes'
>>> ys
{1.0: 'yes'}
```

当然这个作为性能优化来说是有意义的 --- 如果键被认为是相同的，那么为什么要花时间更新原来的？在最开始的例子中，你也可以看到最初的`True`对象一直都没有被替换。因此，字典的字符串表示仍然打印为以`True`为键（而不是 1 或 1.0）。

就目前我们所知而言，似乎看起来像是，结果中字典的值一直被覆盖，只是因为他们的键比较后相等。然而，事实上，这个结果也不单单是由`__eq__`比较后相等就得出的。

## 等等，那哈希值呢？

python 字典类型是由一个哈希表数据结构存储的。当我第一次看到这个令人惊讶的字典表达式时，我的直觉是这个结果与散列冲突有关。

哈希表中键的存储是根据每个键的哈希值的不同，包含在不同的“buckets”中。哈希值是指根据每个字典的键生成的一个固定长度的数字串，用来标识每个不同的键。（[哈希函数详情](https://zh.wikipedia.org/wiki/%E6%95%A3%E5%88%97%E5%87%BD%E6%95%B8)）

这可以实现快速查找。在哈希表中搜索键对应的哈希数字串会快很多，而不是将完整的键对象与所有其他键进行比较，来检查互异性。

然而，通常计算哈希值的方式并不完美。并且，实际上会出现不同的两个或更多个键会生成相同的哈希值，并且它们最后会出现在相同的哈希表中。

如果两个键具有相同的哈希值，那就称为*哈希冲突(hash collision)*，这是在哈希表插入和查找元素时需要处理的特殊情况。

基于这个结论，哈希值与我们从字典表达中得到的令人意外的结果有很大关系。所以让我们来看看键的哈希值是否也在这里起作用。

我定义了这样一个类来作为我们的测试工具：

```python
class AlwaysEquals:
     def __eq__(self, other):
         return True

     def __hash__(self):
         return id(self)
```

这个类有两个特别之处。

第一，因为它的`__eq__`魔术方法（_译者注：双下划线开头双下划线结尾的是一些 Python 的“魔术”对象_）总是返回 true，所以这个类的所有实例和其他任何对象都会恒等：

```python
>>> AlwaysEquals() == AlwaysEquals()
True
>>> AlwaysEquals() == 42
True
>>> AlwaysEquals() == 'waaat?'
True
```

第二，每个`Alwaysequals`实例也将返回由内置函数`id()`生成的唯一哈希值值：

```python
>>> objects = [AlwaysEquals(),
               AlwaysEquals(),
               AlwaysEquals()]
>>> [hash(obj) for obj in objects]
[4574298968, 4574287912, 4574287072]
```

在 CPython 中，`id()`函数返回的是一个对象在内存中的地址，并且是确定唯一的。

通过这个类，我们现在可以创建看上去与其他任何对象相同的对象，但它们都具有不同的哈希值。我们就可以通过这个来测试字典的键是否是基于它们的相等性比较结果来覆盖。

正如你所看到的，下面的一个例子中的键不会被覆盖，即使它们总是相等的：

```python
>>> {AlwaysEquals(): 'yes', AlwaysEquals(): 'no'}
{ <AlwaysEquals object at 0x110a3c588>: 'yes',
  <AlwaysEquals object at 0x110a3cf98>: 'no' }
```

下面，我们可以换个思路，如果返回相同的哈希值是不是就会让键被覆盖呢？

```python
class SameHash:
    def __hash__(self):
        return 1
```

这个`SameHash`类的实例将相互比较一定不相等，但它们会拥有相同的哈希值 1：

```python
>>> a = SameHash()
>>> b = SameHash()
>>> a == b
False
>>> hash(a), hash(b)
(1, 1)
```

一起来看看 python 的字典在我们试图使用`SameHash`类的实例作为字典键时的结果：

```python
>>> {a: 'a', b: 'b'}
{ <SameHash instance at 0x7f7159020cb0>: 'a',
  <SameHash instance at 0x7f7159020cf8>: 'b' }
```

如本例所示，“键被覆盖”的结果也并不是单独由哈希冲突引起的。

## Umm..好吧,可以得到什么结论呢?

检查 python 字典对象中两个 key 是否相同的条件是：**二者的相等性(\_\_eq\_\_)以及 hash 值对比(\_\_hash\_\_)是否相等**。那我们就来总结一下上述讨论的结果：

`{true：'yes'，1：'no'，1.0：'maybe'}` 字典表达式计算结果为 `{true：'maybe'}`，是因为键 `true`，`1` 和 `1.0` 都是相等的，并且它们都有相同的哈希值：

```python
>>> True == 1 == 1.0
True
>>> (hash(True), hash(1), hash(1.0))
(1, 1, 1)
```

也许并不那么令人惊讶，这就是我们为何得到这个结果作为字典的最终结果的原因：

```python
>>> {True: 'yes', 1: 'no', 1.0: 'maybe'}
{True: 'maybe'}
```

我们在这里涉及了很多方面内容，而这个特殊的 python 技巧起初可能有点令人难以置信 --- 所以我一开始就把它比作是`Zen kōan`。

如果很难理解本文中的内容，请尝试在 Python 交互环境中逐个去检验一下代码示例。你会收获一些关于 python 深处知识。

> **注：转载请保留下面的内容**
>
> 原文链接：[https://dbader.org/blog/python-mystery-dict-expression](https://dbader.org/blog/python-mystery-dict-expression)
>
> 译文链接：[http://vimiix.com/post/2017/12/28/python-mystery-dict-expression/](http://vimiix.com/post/2017/12/28/python-mystery-dict-expression/)
