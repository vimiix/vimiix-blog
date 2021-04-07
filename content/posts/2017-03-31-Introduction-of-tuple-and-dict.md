---
title: Python|对于元组和字典的认识
date: 2017-03-31 15:00:06
tags:
  - tuple
  - dict
  - Python
categories: Python
---

本篇是一篇随手笔记，记录了对于 Python 的数据类型中元组（Tuple）和字典（Dict）的一些认识，以及部分内置方法的介绍。

<!--more-->

## 元组 Tuple

##### 特点：元组内的数据不可变

##### 一个元素的定义：T = （1，）

```Python
	>>> T=(1,)
	>>> type(T)
	<type 'tuple'>
```

##### 特殊的元组："可变"的元组

```Python
	>>> T=(1,2,3,[1,2,3])
	>>> T[3][2] = 'vimiix'
	>>> T
	(1, 2, 3, [1, 2, 'vimiix'])
```

看上去元组发生了变化，但真正变化的是<font color=blue>[1，2，3]</font>这个列表内的元素发生了变化，但是这个列表在 T 这个元组中的内存地址是没有改变的。

结论：实际是元组的元素包含了可变的元素，但是元组中元素的内存地址没有变，所以所谓的元组不可变是指元素指向的内存地址是不变

<hr>

## 字典 Dict

##### 特点：

##### 1、字典是 Python 中唯一的映射类型

##### 2、字典的键（KEY）必须是不可变的对象--->因为字典在计算机中是通过 Hash 算法存储的，Hash 的特点是由 KEY 来计算存储的，如果 KEY 可变，将会导致数据混乱。

```Python
	>>> D = {1:3,'vimiix':88}
	>>> type(D)
	<type 'dict'>
```

```Python
	>>> D={[1,2,3]:100}
	Traceback (most recent call last):
	  File "<pyshell#15>", line 1, in <module>
	    D={[1,2,3]:100}
	TypeError: unhashable type: 'list' (这里提示list是不能被Hash计算的数据类型，因为list是可变的数据类型)
	>>>
```

由此错误可以看出，字典的键只能使用不可变的对象（元组是可以的），但是对于字典的值没有此要求
键值对用冒号‘：’分割，每个对之间用逗号‘，’分开，所有这些用花括号‘{}’包含起来
字典中的键值对是没有顺序的，故不可以用索引访问，只可以通过键取得所对应的值

##### 拓展：如果定义的过程中，出现相同的键，最后存储的时候回保留最后的一个键值对）

```Python
	>>> D= {1:2,1:3}
	>>> D
	{1: 3}
```

### 创建与访问

第一种创建方式：直接通过花括号包含键值对来创建

第二种创建方式：利用内置函数 dict()来创建，注意！dict（）括号内只能有一个参数，要把所有的键值对括起来

（1）

```Python
	>>> D =dict((1,2),(3,4),(5,6))
	Traceback (most recent call last):
	  File "<pyshell#20>", line 1, in <module>
	    D =dict((1,2),(3,4),(5,6))
	TypeError: dict expected at most 1 arguments, got 3
	>>> D =dict(((1,2),(3,4),(5,6)))
	>>> D
	{1: 2, 3: 4, 5: 6}
```

（2）还可以指定关键字参数

```Python
	>>> D=dict(vimiix = 'VIMIIX')
	>>> D
	{'vimiix': 'VIMIIX'}
```

这里的小写‘vimiix’不可以加单引号，加了会报错！

（3）dict 的内置方法 .fromkeys 有两个参数

```Python
	>>> D = dict.fromkeys((1,'vimiix'),('common','value'))
	>>> D
	{1: ('common', 'value'), 'vimiix': ('common', 'value')}
	>>>
```

实际的生产过程中，都是使用<font color=blue>字典生成式</font>来创建，根据现有的数据来生成对应的数据，有数据才有意义。

字典生成式栗子：

```Python
	>>> L1 = [1,2,3]
	>>> L2 = ['a','v','vimiix']
	>>> D={a:b for a in L1 for b in L2}
	>>> D
	{1: 'vimiix', 2: 'vimiix', 3: 'vimiix'}
```

###### [???]此处只是一个生成式的栗子，但并不是理想答案，待学习如何生成一一对应的键值对

### 字典的内置方法：

##### <font color=blue>get()</font>:

获取键所对应的值，如果未找到返回 None，找到返回对应的值

##### <font color=blue>pop(key)</font>:

弹出 key 对应的值，默认最后一个

##### <font color=blue>popitem()</font>:

随机返回并删除字典中的一对键和值（项）。为什么是随机删除呢？因为字典是无序的，没有所谓的“最后一项”或是其它顺序。在工作时如果遇到需要逐一删除项的工作，用 popitem()方法效率很高。

##### <font color=blue>update()</font>:

更新或者新增一个键值对（有则改之无则加勉）

```Python
	>>> D.update({'newitem':'update'})
	>>> D
	{'newitem': 'update', 1: 'vimiix', 2: 'vimiix', 3: 'vimiix'}
```
