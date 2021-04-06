---
title: '[译]Python3:变量注释'
date: 2017-11-02 13:12:48
categories: 'translation'
tags: [Python, annotation, 'translation']
---

Python 在 3.6 版中添加了一个叫做**变量注释**的语法。变量注释简单讲就是对于类型提示的增强，这个概念是在 Python3.5 中开始引入的。变量注释的完整解释在[PEP 526](https://www.python.org/dev/peps/pep-0526)中进行了详细说明。在本文中，我们将将要的回顾一下类型提示，然后再介绍新的变量注释语法。

<!--more-->

## 什么是类型提示？

Python 中的类型提示就是明显的声明函数和方法中的参数是具有确定的类型的。 Python 本身是不强制编程时候定义参数的类型“提示”，但是你可以使用像[mypy](http://mypy-lang.org/)这样的工具来强制类型提示，就像 C++一样。我们来看一个常见的没有添加类型提示的函数：

```python
def add(a, b):
    return a + b

if __name__ == '__main__':
    add(2, 4)
```

这里我们创建一个需要两个参数的 add（）函数。它传递两个参数进去并返回结果。但是，我们不明确的是我们应该传什么样的参数给这个函数。我们可以传整形，浮点型，列表，或者字符串进去，看样子它都是可以运行的，但它会按照我们预期的方式工作吗？下面我们来向代码添加一些类型提示：

```python
def add(a: int, b: int) -> int:
    return a + b

if __name__ == '__main__':
    print(add(2, 4))
```

在这里我们通过利用类型提示来改变函数的定义。你将注意到，每一个参数都被标注了它应该是什么类型的：

- `a`:int
- `b`:int

我们还通过函数尾部的`-> int`声明了这个函数的返回值。这仅仅意味着我们期望这个函数返回值是一个整形。但是如果你尝试使用几个字符串或一个浮点型（float）和整数来调用`add（）`函数，你也不会看到错误。就像我前面提到，python 仅仅是允许你去声明一个参数期望的类型，但并不会强制要求这样做。

我们来继续更新一下代码：

```python
def add(a: int, b: int) -> int:
    return a + b

if __name__ == '__main__':
    print(add(5.0, 4))
```

如果你运行这段代码，你会看到它完全可以正常执行。现在让我们使用 pip 安装`mypy`：

```python
pip install mypy
```

现在有了 mypy，我们可以用它来判断我们是否正确调用了我们的函数。打开一个终端，并切换目录到保存前面写的脚本的文件夹下。然后用下面的命令来执行一下：

```python
mypy hints.py
```

当运行了这个指令，终端返回了这些信息：

```python
hints.py:5: error: Argument 1 to "add" has incompatible type "float";
expected "int"
```

可以看到，mypy 发现我们的代码有问题。我们传递一个浮点数作为第一个参数，而不是函数所期望的`int`。

你可以在一些持续集成服务器上使用 mypy，这样我们就可以在将代码提交到分支之前检查好我们的代码，或者在提交代码之前在本地运行检查一下。

---

## 变量注释

假设你不仅要声明函数参数的类型，还要定义常量变量的类型。在 Python 3.5 中，你是无法使用与函数参数类型提示这样相同的语法来声明常量或者变量，因为它会引发`SyntaxError`（语法错误）。代替方案只能是对于常量或者变量添加注释。但是现在，python3.6 发布了，我们可以使用新的语法实现这个功能。我们一起来看一个例子：

```python
from typing import List

def odd_numbers(numbers: List) -> List:
    odd: List[int] = []
    for number in numbers:
        if number % 2:
            odd.append(number)

    return odd

if __name__ == '__main__':
    numbers = list(range(10))
    print(odd_numbers(numbers))
```

这段代码中指定了一个变量`odd`,显式的声明了它应该是整数列表。如果你使用 mypy 来执行这个脚本，将不会收到任何提示输出，因为我们正确地传递了期望的参数去执行所有操作。下面，让我们尝试更改一下代码，试图去添加整形之外的其他类型内容！

```python
from typing import List

def odd_numbers(numbers: List) -> List:
    odd: List[int] = []
    for number in numbers:
        if number % 2:
            odd.append(number)

    odd.append('foo')

    return odd

if __name__ == '__main__':
    numbers = list(range(10))
    print(odd_numbers(numbers))
```

这里我们在代码中添加一个行新代码，将一个字符串`foo`附加到整数列表中。现在，如果我们针对这个版本的代码来运行 mypy，我们应该看到以下内容：

```python
hints2.py:9: error: Argument 1 to "append" of "list" has incompatible type "str"; expected "int"
```

再次重申一下，在 Python 3.5 中，你需要做变量声明，但是你必须将声明放在注释中：

```python
# Python 3.6
odd: List[int] = []

# Python 3.5
odd = [] # type: List[int]
```

请注意，如果您更改代码以使用 Python 3.5 的变量注释语法，mypy 仍将正确标记该错误。你必须在`#`井号之后指定`type：`。如果你删除它，那么它就不再是变量注释了。基本上**PEP 526**增加的所有内容都为了使语言更加统一。

---

## 总结

基于本文，相信你应该对于变量注释已经有足够的认识，那么现在就开始在你自己的代码中使用变量注释吧，无论你使用的是 Python 3.5 还是 3.6。我认为这是一个代码整洁的概念，如果我们需要和那些更熟悉静态类型语言的开发者们一起开发的话，这样做会更加有帮助。

## 相关知识

- Python 3: [An Intro to Type Hinting](https://www.blog.pythonlibrary.org/2016/01/19/python-3-an-intro-to-type-hinting/)
- PEP 526 — [Syntax for Variable Annotations](https://www.python.org/dev/peps/pep-0526/)

---

> 翻译：[vimiix](www.vimiix.com)
>
> 原文链接：[Python 3: Variable Annotations](http://www.blog.pythonlibrary.org/2017/10/31/python-3-variable-annotations/)
>
> 版权归原作者所有，如有冒犯，联系删除
