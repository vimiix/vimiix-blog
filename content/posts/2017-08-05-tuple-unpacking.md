---
title: 'Python|元组拆包和具名元组解析'
date: 2017-08-05 17:02:29
tags:
  - Python
  - tuple
  - namedtuple
categories: Python
---

## 前言

在 Python 中元组是一个相较于其他语言比较特别的一个内置序列类型。有些 python 入门教程把元组成为“不可变的列表”，这种说法是不完备的，其并没有完整的概括元组的特点。除了用作不可变的列表，它还可以用于没有字段名的数据记录。下面的内容就围绕元组作为数据记录属性展开，并介绍带字段名的具名元组函数`namedtuple`，列表属性不再本文中叙述。

<!--more-->

## 元组对于数据的记录

元组中的每个元素都存放了记录中一个字段的数据，外加这个字段的位置，正是这个位置信息给数据赋予了意义。

下面的一段代码就演示了元组被当作记录来使用。如果在任何的表达式里我们在元组内对元素排序，这些元素多携带的信息就会丢失，因为这些信息是跟它们的位置强关联的。

```Python
	#把元组作记录
	>>> xiaoming, xiaohua = (16, 18)
	>>> xiaoming
	16
	>>> students_info = [('xiaoming', 16), ('xiaohua', 18), ('hanmeimei', 20)]
	>>> for student in students_info:
		print('%s is %d years old.'%student)

	xiaoming is 16 years old.
	xiaohua is 18 years old.
	hanmeimei is 20 years old.
	>>>
```

在这个示例中，我们把元组`（16，18）`里的元素分别赋值给变量`xiaoming`,`xiaohua`。同样在 for 循环中，一个`%`运算符就把`student`元组里的元素对应到了 Print 函数的格式字符串空档中。这两个都是**元组拆包**的应用。

元组拆包可以应用到任何可迭代对象上，唯一的硬性要求是，被可迭代对象中的元素数量必须要跟接受这些元素的元组的空档数一致。除非用`*`来表示忽略多余的元素。

## 元组拆包

最好辨认的元组拆包形式就是**平行赋值** ，也就是把一个可迭代对象里的元素，一并赋值到由对应的变量组成的元组中。例如：

```Python
	>>> age_list = (16,18)
	>>> xiaoming, xiaohua = age_list #这里就是元组拆包
```

另一个我们熟悉的平行赋值的例子就是交换两个变量的值：

```Python
	>>> a, b = b, a #Python就是如此的优雅
```

还可以用`*`运算符把一个可迭代对象拆开作为函数的参数：

```Python
	>>> divmod(20,8)
	(2, 4)
	>>> t = (20, 8)
	>>> divmod(*t)
	(2, 4)
	>>> quotient, remainder = divmod(*t)
	>>> quotient, remainder
	(2, 4)
```

### 用\*来处理剩下的元素

在 Python 中，函数用\*args 来获取不确定数量的参数算是一种经典写法了。在 Python3 中，这个概念被扩展到了平行赋值中：

```Python
	>>> a, b, *rest = range(5)
	>>> a, b, rest
	(0, 1, [2, 3, 4])
	>>> a, b, *rest = range(3)
	>>> a, b, rest
	(0, 1, [2])
	>>> a, b, *rest = range(2)
	>>> a, b, rest
	(0, 1, [])
```

在平行赋值中，`*`运算符前缀智能用在一个变量名前面，但是这个变量可以出现在赋值表达式的任意位置：

```Python
	>>> a, *others, b, c = range(5)
	>>> a, others, b, c
	(0, [1, 2], 3, 4)
	>>> *others, a, b, c = range(5)
	>>> others, a, b, c
	([0, 1], 2, 3, 4)
```

## 具名元组

在 Python 中，`collections.namedtuple`是一个工厂函数，它可以用来构建一个带字段名的元组和一个有名字的类。

用`namedtuple`构建的类的实例所消耗的内存跟元组是一样的，因为字段名都被存在对应的类里面。这个实例跟普通的对象实例比起来也要小一些，因为 python 不会用`__dict__`来存放这些实例的属性。

还是使用上面的小明和小华的例子来展示一下具名元组：

```Python
	>>> from collections import namedtuple
	>>> Student = namedtuple('Student', 'name age gender')
	>>> xiaoming = Student('xiaoming', 16, 'boy')
	>>> xiaoming
	Student(name='xiaoming', age=16, gender='boy')
	>>> xiaoming.age
	16
	>>> xiaoming[2]
	'boy'
```

`Student = namedtuple('Student', 'name age gender')`，创建一个具名元组，需要两个参数，一个是类名，另一个是类的各个字段名。后者可以是有多个字符串组成的可迭代对象，或者是有空格分隔开的字段名组成的字符串（_比如本示例_）。具名元组可以通过字段名或者位置来获取一个字段的信息。

##### 具名元组的特有属性

- 类属性**`_fields`**：包含这个类所有字段名的元组

```Python
	>>> xiaoming._fields
	('name', 'age', 'gender')
```

- 类方法**`_make(iterable)`**：接受一个可迭代对象来生产这个类的实例，作用等价于`Student(*xiaohua_info)`

```Python
	>>> xiaohua_info = ('xiaohua', 18, 'girl')
	>>> xiaohua = Student._make(xiaohua_info)
	>>> xiaohua
	Student(name='xiaohua', age=18, gender='girl')
```

- 实例方法**`_asdict()`**：把具名元组以`collections.OrdereDict`的形式返回，可以利用它来把元组里的信息友好的展示出来

```Python
	>>> xiaohua._asdict()
	OrderedDict([('name', 'xiaohua'), ('age', 18), ('gender', 'girl')])
	>>> for key, value in xiaohua._asdict().items():
		print(key,':',value)

	name : xiaohua
	age : 18
	gender : girl
```
