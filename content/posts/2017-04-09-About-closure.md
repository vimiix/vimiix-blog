---
title: 'Python-关于Python中闭包的一些理解'
date: 2017-04-09 17:07:34
tags:
  - closure
  - Python
categories: Python
---

![](https://static.vimiix.com/uPic/2021-04-06/rRo70V.jpg)

**看不懂的定义**：闭包是由函数及其相关的引用环境组合而成的实体(即：闭包=函数+引用环境)。

<!--more-->

既然是看不懂的定义，真看不懂上面定义的话就忽略吧。

在 python 中，函数可以作为另一个函数的参数或返回值，可以赋给一个变量。函数可以嵌套定义，即在一个函数内部可以定义另一个函数，有了嵌套函数这种结构，便会产生闭包问题。

**好理解一点的定义**：如果在一个内部函数里，对在外部作用域（但不是在全局作用域）的变量进行引用，那么内部函数就被认为是闭包(closure)

举个栗子：

```Python
	def outer(x):
		def inner(y):
			return x + y
		return inner
```

结合代码分析定义：

<font color=green>如果在一个内部函数里</font> --- inner()就是内部函数。

<font color=green>对在外部作用域（但不是在全局作用域）的变量进行引用</font> --- x 就是被引用的变量，x 在外部作用域，但不在全局作用域。

<font color=green>那么内部函数就被认为是闭包</font> ---- inner 就是一个闭包。

### 关于闭包很难理解的一个问题，我尝试用图形化思维来理解

先看一个简单的循环

```Python
	for i in range(3):
		print i
```

在程序里面经常会出现这类的循环语句，python 的一个现象是，当循环结束以后，循环体中的临时变量 i 不会销毁，而是继续存在于执行环境中。还有一个 python 的现象是，python 的函数只有在执行时，才会去找函数体里的变量的值。

> 这段话特别需要记住两点：
>
> 1. 当循环结束时，循环体中的临时变量 i 不会销毁
> 2. python 的函数只有在执行时，才会去找函数体里的变量的值

记住上面两点后，下面看经典的难理解的栗子：

```Python
	def foo():
		func_list = []
		for i in range(3):
			def inner():
				return i*i
			func_list.append(inner)
		return func_list

	f = foo()
```

在这个例子中，每次循环都创建一个新的函数，并且将创建的三个<font color=red>函数对象</font>都添加到<font color=red>func_list</font>这个列表中

<font color=green>f = foo()</font>这里调用 foo()，f 中就保存了一个列表对象，这个列表中保存了 3 个函数对象。

不妨打印一下看看 <font color=red>**f**</font> 中三个元素的值：

```Python
	>>> print f[0],'\n',f[1],'\n',f[2]
	<function inner at 0x000000000263BAC8>
	<function inner at 0x0000000002E664A8>
	<function inner at 0x0000000002E66518>
```

从打印信息可以看出，<font color=red>**f**</font> 中存放了 3 个函数名相同，但内存地址不同，的函数对象。

此时调用一下三个函数

```Python
	>>> f[0]()
	4
	>>> f[1]()
	4
	>>> f[2]()
	4
```

可能有些人认为这段代码的执行结果应该是 0,1,4.但是实际的结果是 4,4,4。这是因为当把函数对象加入 func_list 列表里时，python 还没有给 i 赋值，只有当执行时，再去找 i 的值是什么，这时在第一个 for 循环结束以后，i 的值是 2，所以以上代码的执行结果是 4,4,4.

不好理解的话，画个流程图(<font color=grey>点击图片查看大图</font>)：

![流程图](https://static.vimiix.com/uPic/2021-04-06/SzLD2R.png)

这里也可以直观的理解文章开始提到的闭包的定义公式（<font color=red>闭包=函数+引用环境</font>）

结果全部都是 <font color=red>4</font>，原因就在于返回的函数引用了变量 i，但它并非立刻执行。等到 3 个函数被调用时，它们所引用的变量 i 已经变成了 2，因此最终结果为 4。

返回闭包时牢记的一点就是：<font color=red>返回函数不要引用任何循环变量，或者后续会发生变化的变量</font>。

如果要正确的输出引用循环变量后的值，只需要将每次循环变量锁定到闭包中，具体实现如下：

```Python
	def foo():
	      func_list = []
	      for i in range(3):
	            def inner( x = i):
	                  print x*x
	            func_list.append(inner)
	      return func_list

	f = foo()
```

这样的话，打印调用每个闭包后的结果为<font color=red> 0 , 1, 4 </font>

```Python
	>>> f[0]()
	0
	>>> f[1]()
	1
	>>> f[2]()
	4
```
