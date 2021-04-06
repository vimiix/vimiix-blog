---
title: '[译]python中垃圾回收和析构函数笔记'
date: 2018-06-04 14:36:48
categories: 'Python'
tags: ['GC', 'Python']
---

> 紧接上一篇转载的文章[《Python 魔术方法总结》](https://vimiix.com/post/2018/06/04/python-magic-methods/)文末提及的英文参考文章，洒家顺手就翻译了一下。方便墙内的同学学习。这篇文章不仅指出了 Python 如何处理垃圾回收，还提到了我们作为程序员不应该只借助现代化 IDE 的函数提示功能去完成代码，应该多去看官方的完整文档，可以知道哪些函数被废弃的，哪些函数在使用时需要注意什么等等一些很重要的信息。话不多说，自己体会，请向下阅读文章吧。
>
> 原文链接：[https://www.electricmonk.nl/log/2008/07/07/python-destructor-and-garbage-collection-notes/](https://www.electricmonk.nl/log/2008/07/07/python-destructor-and-garbage-collection-notes/)

我很少在 Python 对象中使用析构函数。我猜 Python 的动态特性往往弱化了对析构函数的需求。但是现在，假如我需要在对象被销毁时，或者更确切地说，当程序退出时，要将一些数据写入磁盘。这时我会使用`__del__` 魔术方法在主要操作的类对象中定义了一个析构函数。但是奇怪的是，这个析构函数自始至终都没有被调用到。不仅在程序退出时没有被调用到，而且我手动使用`del`删除时也不会被调用。由于这个程序是我前一段时间写的，所有稍微有点不是很熟悉了，这导致我怀疑是我程序中有一个大的 BUG 存在。

<!--more-->

最后我终于一点一点将问题追溯到类似下面这些基本代码所做的事情一样：

```python
class Foo:
	def __init__(self, x):
		print "Foo: Hi"
		self.x = x
	def __del__(self):
		print "Foo: bye"

class Bar:
	def __init__(self):
		print "Bar: Hi"
		self.foo = Foo(self) # x = 这个实例

	def __del__(self):
		print "Bar: Bye"

bar = Bar()
# del bar # 这一行也不起作用
```

上面的代码做的是实例化一个`Bar`的一个实例来保持`Foo`的实例保持对其创建者类的引用。代码输出如下：

```python
Bar: Hi
Foo: Hi
```

正如你所看到的，析构函数从来没有被调用过，即使我们在程序的最后添加了一个`del`主动删除。删除`self.x = x`解决了（好吧，让它消失）这个问题。

## 垃圾回收

查看上面的代码时，`__del__`从未被调用的原因突然变得明显。对于某些垃圾回收器来说，这是一个'问题'，即：**循环引用**。Python 使用引用计数垃圾回收算法。这种垃圾回收算法会为存在该数据实例的每个引用的每个数据实例增加一个计数器，并在删除对数据实例的引用时减少计数器。当计数器达到`0`时，数据实例被垃圾回收，因为没有任何对象指向它了。引用计数存在循环引用问题。在上面的代码中，`foo.x`指向`bar`，`bar.foo`指向`foo`。这意味着引用计数器永远不会下降，并且对象永远不会被垃圾回收。因为这个，析构函数永远不会被调用。

`del foo`不起作用的原因也很容易解释。我最初疑惑是因为我以为`del foo`会直接去调用析构函数，但实际上它只是会减少对象的引用计数器（并从本地作用域中删除引用）。上面代码中由于`Foo`和`Bar`的计数是`2`（主程序中的一个，其他实例中的一个），所以对象的计数值只会下降到`1`。

## 更多信息

弄清楚了这一点后，有人（这里要感谢 Cris）向我指出了关于`__del__`的文档中的说明。如果我之前就读过它，我会注意到那里的笔记：

> `del x`不直接调用`x .__ del __()` - 前者将`x`的引用计数递减 1，后者仅在`x`的引用计数达到 0 时调用。一些常见情况可能会阻止对象的引用计数从零到包括：对象之间的循环引用
>
> （原文）"del x" doesn't directly call `x.__del__()` — the former decrements the reference count for x by one, and the latter is only called when x's reference count reaches zero. Some common situations that may prevent the reference count of an object from going to zero include: circular references between objects

除此之外，我在`__del__`的文档中还注意到了一些其他内容：

> 当选项循环检测器被启用时（它默认为开启），会检测到垃圾循环引用，但只有在没有涉及 Python 级别的`__del__()`方法时才能清除。
>
> （原文）Circular references which are garbage are detected when the option cycle detector is enabled (it's on by default), but can only be cleaned up if there are no Python-level `__del__()` methods involved.

进一步阅读表明：

> A list of objects which the collector found to be unreachable but could not be freed (uncollectable objects). By default, this list contains only objects with **del**() methods.26.1Objects that have **del**() methods and are part of a reference cycle cause the entire reference cycle to be uncollectable, including objects not necessarily in the cycle but reachable only from it. Python doesn't collect such cycles automatically because, in general, it isn't possible for Python to guess a safe order in which to run the **del**() methods. […] It's generally better to avoid the issue by not creating cycles containing objects with **del**() methods

这意味着带有循环引用和`__del__`方法的对象将在你的 Python 程序中产生内存泄漏，除非在对象被删除之前手动中断循环引用。有些事情仍是值得需要考虑的。

## 程序退出

您可能想知道为什么 Python 在退出程序时不会简单地将所有引用计数设置为 0？正如 BDFL([Guido 先生](https://gvanrossum.github.io//))在有篇文章中关于`__del__`所述：

> 最后一件需要思考的事情是：如果我们有`__del__`方法，解释器是否应该保证在程序退出时调用它？（就像 C ++一样，它保证全局变量的析构函数会被调用。）保证这一点的唯一方法是遍历所有模块并删除所有变量。但这同样也意味着`__del__`方法不确定在它方法内想要使用的任何全局变量是否仍然存在（译者注：也许在调用`__del__`函数之前，要用到的全局变量已经被释放掉了），因此我们无法知道要以何种顺序删除变量。
>
> （原文）One final thing to ponder: if we have a **del** method, should the interpreter guarantee that it is called when the program exits? (Like C++, which guarantees that destructors of global variables are called.) The only way to guarantee this is to go running around all modules and delete all their variables. But this means that **del** method cannot trust that any global variables it might want to use still exist, since there is no way to know in what order variables are to be deleted.

像手册中提到的一样：不能保证在解释器退出时对象`__del__()`方法所要析构的对象是否仍然存在。

## 析构函数内的异常

正如 Guido 先生在帖子中提到的那样，在析构函数中引发的异常会被忽略：

```python
    def __del__(self):
            raise Exception("Oopsy")
            print "Bar: Bye"
Exception exceptions.Exception: Exception('Oopsy',) in > ignored
Bar: Hi
Foo: Hi
Foo: bye
```

（该警告是在编译期间生成的，而不是在运行时生成的）

## 更多备注

另一方面：这就是学生为什么要学习软件工程专业的基本知识，如 C 程序以及垃圾收集器和各种算法如何工作的原因，尽管一些教育工作者似乎认为现在的高级语言不再需要这些信息。

此外，请不要完全依赖 IDE 对方法的简短描述提示来完成代码。尽管我也可能不会阅读关于`__del__`和`del`的完整文档，但我经常发现，如果不阅读方法的完整文档（特别是一些弃用声明，安全问题和特别令人讨厌的副作用），则会错过一些重要注释和限制提示。

## 更新

防止循环引用的好方法应该是`weakref`模块：[weakref - 弱引用](http://docs.python.org/lib/module-weakref.html)。快速介绍：[Mindtrove：Python 弱引用](http://mindtrove.info/articles/python-weak-references/)。（感谢`zzzeek @ reddit`指出`weakrefs`）

— EOF —
