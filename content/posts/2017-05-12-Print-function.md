---
title: 'Python-from __future__ import print_function'
date: 2017-05-12 14:55:03
tags:
  - future module
  - Python
categories: Python
---

Python 提供了**\_\_future\_\_**模块，把下一个新版本的特性导入到当前版本，于是我们就可以在当前版本中测试一些新版本的特性

在 python2.x 的环境是使用下面语句，则第二句语法检查通过，第三句语法检查失败

```Python
	from __future__ import print_function
	print('good')  #可以通过执行
	print 'bad'    #语法错误
```

<!--more-->

在 python2.x 中，默认的**print**只是一个简单的输出流方法，不带有任何的参数。

用下面的两个例子概况：

**示例 1**

```Python
	var, var1, var2 = 1, 2, 3
	print var
	print var1, var2
```

示例 1 会打印两行，在第二行中两个数字之间会有一个空格。

**示例 2**

```Python
	for i in xrange(10):
		print i,
```

示例 2 会将每个数字打印在一行上，每个数字之间有一个空格分割。如果去掉打印时 i 后面的“,”,每个数字会单独占一行。

下面从**\_\_future\_\_**众引入对于 python2 来说比较先进的模块**print_function**

接口参数：

```Python
	print(*values, sep=' ', end='\n', file=sys.stdout)
	print(value1, value2, value3, sep=' ', end='\n', file=sys.stdout)
```

这里，输出的变量可以是一个序列或者多个变量，可以像上面一样用逗号分开每个变量。 参数`sep`,`end`,`file`是三个可选参数。

`sep` 指每个输出变量之间的分隔符，默认是一个空格

`end` 指的是输出结束后的内容，默认是换行

`file`指的是输出流要输出的目的文件，默认 sys.stdout（标准输出）

在 Pyhton2 中，`print_function` 比默认的 `print`效率要快很多！
