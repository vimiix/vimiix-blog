---
title: 'Python|内建函数getattr()函数详解'
date: 2017-07-10 12:08:03
tags:
  - Python
  - getattr
categories: Python
---

## 前言

昨天在[Hackerrank](https://www.hackerrank.com)上面练习 Python 题目的时候，题目要求从终端输入命令和参数，然后脚本获取到以后执行相应的操作（[查看原题](https://www.hackerrank.com/challenges/py-set-discard-remove-pop)），作为初级程序员，我第一思路想到的解法就是枚举（因为题目上面限制了就 3 条命令）。通过`if ... elif...elif...`的分支语句就解决了。Hackerrank 有一个好处就是，每一个题目都有一个讨论区，在讨论区中往往会有超出本人能力范围的巧妙高级解法发表在里面。有个大牛在讨论区发表了自己的解法，其中就用到了 getattr 这个内建函数，这样代码的扩展性就极强了，不再局限于本题目的三条指令。下面就记录一下这个有趣的函数。

> [上述代码查看传送门](http://t.cn/RKtqwu4)

<!--more-->

## 记录

getattr 是 python 里的一个内建函数,先看定义

```Python
	>>> help(getattr)
	Help on built-in function getattr in module __builtin__:

	getattr(...)
	    getattr(object, name[, default]) -> value

	    Get a named attribute from an object; getattr(x, 'y') is equivalent to x.y.
	    When a default argument is given, it is returned when the attribute doesn't
	    exist; without it, an exception is raised in that case.

	>>>
```

getattr()的三个参数：

- object : 要实例化的对象
- name ： 要执行的命令操作
- default :（可省略参数）当要执行的命令操作不存在是，会返回这里传入的 default

getattr()主要的作用是通过传入字符串的方式，来动态的获取方法的实例。这个函数最大的用处在于函数解耦，可以讲一些可能会调用的方法，写到一个配置文件中，在需要的时候动态加载。

解释的比较抽象，简单来说：`getattr(object, name)` 等价于 `object.name`

下面通过代码说明一下

```Python
	for i in range(t):
	    c, *args = map(str,input().split())
	    getattr(s,c) (*(int(x) for x in args))
```

这段代码实现了用户从`stdin`输入命令和参数，命令赋值给了`c`，参数传给了一个指针变量`*args`,这里使用`*args`是为了接收不确定个数的参数。然后调用`getattr(s,c)`来执行`c`这条指令（注：`s`是一串字符串），此处`getattr(s,c)`就可以等价于`s.c`,后面括号内`(*(int(x) for x in args))`,这里是一个生成器表达式，这部分称为参数解包，它类似于如何定义具有任意数量（位置）参数的函数 --- `*`在序列之前将遍历序列并将其成员变为函数调用参数。

可以看出，这样的用法比枚举的思路方便了很多，也具有更高的扩展性。

## 延伸

Python 中和 getattr 相关的还有`hasattr`、`setattr`、`delattr`，通过举例了解一下。

```Python
	#定义一个类对象
	class Vimiix:
		def __init__(self):
			self.name = 'vimiix'
		def setName(self, name):
			self.name = name
		def getName(self):
			return self.name

	#实例化对象
	foo = Vimiix()
```

##### hasattr(object, name) 判断 object 是否具有 name 属性

```Python
	>>>hasattr(foo,'getName')

	True
```

##### setattr(object,name,default) 设置一个新的属性，并赋予值.类似 foo.age = 18

```Python
	>>>setattr(foo, 'age', '18')
```

##### delattr(object, name) 删除 object 对象的 name 属性

```Python
	>>>delattr(foo, 'age')
	>>>getattr(foo, 'age', 'Not find')

	'Not find'
```
