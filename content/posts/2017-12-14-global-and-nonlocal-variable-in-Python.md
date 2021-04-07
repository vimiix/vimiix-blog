---
title: '[译]python中的global和nonlocal的实践'
date: 2017-12-14 18:51:48
categories: 'translation'
tags: [python, 'translation', global, nonlocal]
---

> 今天的博文翻译是关于 python 中 global 和 nonlocal 两个关键字的用法，原文的作者是来自孟加拉国的[Tamim Shahriar](https://twitter.com/subeen)，他的[博客](http://love-python.blogspot.com.br/?view=classic)非常适合新手朋友去阅读，都是简短而有意义的 python 实践。

我们大多数人都对 Python 中的全局变量很熟悉了。如果我们在一个模块中声明全局变量，模块内部的任何函数都可以访问这个全局变量。（模块可以理解为一个`python`文件或`.py`文件）

<!--more-->

例如下面的代码：

```python
x = 5

def myfnc():
	print("inside myfnc", x)
 	def myfnc2():
		print("inside myfnc2", x)
	myfnc2()

myfnc()
```

这段代码将会输出：

```python
inside myfnc 5
inside myfnc2 5
```

如果我们来改变一下代码，就像下面这样：

```python
x = 5

def myfnc():
	print("inside myfnc", x)
	def myfnc2():
		print("inside myfnc2", x)
		x = 10
		print("x = ", x)

	myfnc2()

myfnc()
```

结果会得到如下错误：

```python
File "program.py", line 6, in myfnc2
    print("inside myfnc2", x)
UnboundLocalError: local variable 'x' referenced before assignment
```

在函数`myfnc2()`中一旦声明了`x = 10`，Python 就会认为`x`是一个局部变量（译者注：此处涉及到 python 的 BGEL 变量优先级原则，可参考[此文章](https://docs.lvrui.io/2016/07/12/Python%E7%9A%84%E5%8F%98%E9%87%8F%E4%BD%9C%E7%94%A8%E5%9F%9F/)了解），而在打印函数时，`x`在声明之前就使用了它，所有它给出了这个错误。因为局部变量是在编译时才确定的（来自官方文档：“事实上，局部变量已经是静态确定了”）(译者注：对于这句话没法直观的理解，可以继续参考[这篇翻译](http://blog.csdn.net/lwl_ls/article/details/1731318))如果将 x 声明是全局的，则可以排除这个错误。

```python
x = 5

def myfnc():
	print("inside myfnc", x)
	def myfnc2():
		global x
		print("inside myfnc2", x)
		x = 10
		print("x = ", x)

	myfnc2()

myfnc()
```

现在你可以再次运行程序，它就不会抛出任何错误。

如果我们现在这样写呢？

```python
x = 5

def myfnc():
	print("inside myfnc", x)
	y = 10
	def myfnc2():
		global x
		print("inside myfnc2", x, y)
		x = 10
		print("x = ", x)

	myfnc2()

myfnc()
```

如果你运行该程序，你会看到正确的输出。但是如果要在`myfnc2（）`中写入 y（例如，指定`y = 1`之类），则不能使用 `global y`，因为 y 不是全局变量。你不妨试试下面这个失败的代码：

```python
x = 5

def myfnc():
	print("inside myfnc", x)
	y = 10
	def myfnc2():
		global x
		global y
		print("inside myfnc2", x, y)
		x = 10
		print("x = ", x)
		y = 1
		print("y = ", y)

	myfnc2()

myfnc()
```

你会得到这个错误：`NameError: name 'y' is not defined`

我们需要明白，`y`不是一个全局变量。这里`nonlocal`就起作用了！仅仅只需写成`nonlocal y`来替换`global y`。它就可以使`myfnc2（）`中的`y`正常读写，调用。

```python
x = 5

def myfnc():
	print("inside myfnc", x)
	y = 10
	def myfnc2():
		global x
		nonlocal y
		print("inside myfnc2", x, y)
		x = 10
		print("x = ", x)
		y = 1
		print("y = ", y)

	myfnc2()

myfnc()
```

这是我今天学到的东西。 :)

> 原文链接：[http://love-python.blogspot.com.br/2017/06/global-and-nonlocal-variable-in-python.html?view=classic](http://love-python.blogspot.com.br/2017/06/global-and-nonlocal-variable-in-python.html?view=classic)
>
> 译文链接：[http://vimiix.com/post/2017/12/14/global-and-nonlocal-variable-in-Python/](http://vimiix.com/post/2017/12/14/global-and-nonlocal-variable-in-Python/)
