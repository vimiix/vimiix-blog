---
title: 'Python-文件操作'
date: 2017-04-13 15:50:34
tags:
  - file
  - Python
categories: Python
thumbnail: https://static.vimiix.com/uPic/2021-04-06/uDCgf4.jpg
---

![header](https://static.vimiix.com/uPic/2021-04-06/uDCgf4.jpg)

## 打开文件

在 Pyhton 中，使用<font color=red>open()</font>函数打开文件，并返回文件对象，拿到这个文件对象，就可以读取或修改这个文件。<font color=red>open()</font>是一个 python 内建函数，可以直接调用。

<!--more-->

```Pyhton
	#open()函数在python2.7的介绍
	Help on built-in function open in module __builtin__:

	open(...)
	    open(name[, mode[, buffering]]) -> file object

	    Open a file using the file() type, returns a file object.  This is the
	    preferred way to open a file.  See file.__doc__ for further information.
```

上面的介绍是在 Python2.7 中的介绍，python3 中新增了很多参数，暂不需要了解。目前只需要关注前两个参数。

第一个参数<font color=blue>name</font>：是传入的文件名，如果只是文件名，不带路径的话，Python 会在当前文件夹中去寻找该文件并打开。

第二个参数<font color=blue>mode</font>：指定文件打开的模式，默认为'r'

<center>文件的打开模式</center>

| 打开模式 |                                                               对应操作                                                               |
| :------: | :----------------------------------------------------------------------------------------------------------------------------------: |
|   'r'    |                                        只读方式打开文件，文件的指针将会放在文件的开头（默认）                                        |
|   'w'    |                              只写方式打开文件，如果该文件已存在则将其覆盖，如果该文件不存在，创建新文件                              |
|   'a'    | 追加方式打开文件。如果该文件已存在，文件指针将会放在文件的结尾，新的内容将会被写入到已有内容之后。如果该文件不存在，创建新文件并写入 |
|   'r+'   |                                           打开一个文件用于读写,文件指针将会放在文件的开头                                            |
|   'w+'   |                             打开一个文件用于读写,如果该文件已存在则将其覆盖,如果该文件不存在，创建新文件                             |
|   'a+'   |   打开一个文件用于读写,如果该文件已存在，文件指针将会放在文件的结尾，文件打开时会是追加模式，如果该文件不存在，创建新文件用于读写    |
|   'b'    |                                                         以二进制模式打开文件                                                         |

示例：

```Python
	f=open('test.txt','w')#以只写方式打开文件test.txt
	f.close() #关闭文件
```

1、执行 open 语句后没有报错，就说明文件被成功的打开了。

2、调用 close()方法关闭文件。文件使用完毕后必须关闭，因为文件对象会占用操作系统的资源，并且操作系统同一时间能打开的文件数量也是有限的

## 文件对象的方法

<center> 常用的几个文件对象方法</center>

|      文件对象方法      |                                         对应操作                                          |
| :--------------------: | :---------------------------------------------------------------------------------------: |
|        close()         |                                         关闭文件                                          |
|      read([size])      |      从文件中读取 size 个字符，没有给定 size 时，读取文件所有字符，并返回一个字符串       |
|       readline()       |                                  读取一行，遇到'\n'停止                                   |
|     readlines（）      |                             一次读取所有内容，并按行返回 list                             |
|       write(str)       |                                   将字符串 str 写入文件                                   |
|    writelines(seq)     |              向文件写入字符串序列 seq，seq 应该是一个返回字符串的可迭代对象               |
| seek(offset[, whence]) | 在文件中移动文件指针，从 whence(0:文件起始位置，1:当前位置，2:文件末尾)偏移 offset 个字节 |

## 读文件操作优化（本段摘自[廖雪峰教程-文件读写](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386820066616a77f826d876b46b9ac34cb5f34374f7a000)）

由于文件读写时都有可能产生**IOError**，一旦出错，后面的**f.close()**就不会调用。

所以，为了保证无论是否出错都能正确地关闭文件，我们可以使用**try ... finally**来实现：

```Python
	try:
	    f = open('/path/to/file', 'r')
	    print f.read()
	finally:
	    if f:
	        f.close()
```

但是每次都这么写实在太繁琐，所以，Python 引入了<font color=red>**with**</font>语句来自动帮我们调用 close()方法：

```Python
	with open('/path/to/file', 'r') as f:
	    print f.read()
```

这和前面的 try ... finally 是一样的，但是代码更佳简洁，并且不必调用 f.close()方法。

调用 read()会一次性读取文件的全部内容，如果文件有 10G，内存就爆了，所以，要保险起见，可以反复调用 read(size)方法，每次最多读取 size 个字节的内容。另外，调用 readline()可以每次读取一行内容，调用 readlines()一次读取所有内容并按行返回 list。因此，要根据需要决定怎么调用。

如果文件很小，read()一次性读取最方便；如果不能确定文件大小，反复调用 read(size)比较保险；如果是配置文件，调用 readlines()最方便：

```Python
	for line in f.readlines():
	    print(line.strip()) # 把末尾的'\n'删掉
```

## 文件的定位 seek(offset[, whence])

当以不同的方式打开文件的时候，文件指针存在的位置是不同的，比如以'w'方式打开文件，文件指针是在文件起始位置，如果以'a'方式打开，文件指针是在文件末尾的。

通过 seek(offset[, whence])方法，就可以控制文件指针的位置，定位在我们需要的地方。

参数 offset 表示从文件中移动 offset 个操作标记（文件指针），正往结束方向移动，负往开始方向移动。whence，就以 whence 设定的起始位为准，0 代表从头开始，1 代表当前位置，2 代表文件最末尾位置。

实例：

```Pyhton
	#coding=utf-8

	# 打开一个foo.txt文件
	f = open("foo.txt", "r+")
	print "Name of the file: ", f.name

	# 假设文件中包含下面5行内容
	# This is 1st line
	# This is 2nd line
	# This is 3rd line
	# This is 4th line
	# This is 5th line

	line = f.readline()
	print "Read Line: %s" % (line)

	# 将文件指针移回起始位置
	f.seek(0, 0)
	line = f.readline()
	print "Read Line: %s" % (line)

	# 关闭文件
	f.close()
```

执行结果：

```Python
	================== RESTART: C:/Users/vimiix/Desktop/test.py ==================
	Name of the file:  foo.txt
	Read Line: This is 1st line

	Read Line: This is 1st line

	>>>
```
