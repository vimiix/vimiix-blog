---
title: "Python-高阶函数习题练习"
date: 2017-04-08 15:50:34
tags:
  - map
  - reduce
  - filter
  - Python
categories: Python
thumbnail: http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/map-reduce-filter.jpg

---

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/map-reduce-filter.jpg)

##### 本文是针对map()，reduce()和filter()三个高阶函数的程序练习。

<!--more-->

习题来自：[廖雪峰的python2.7教程 - map/reduce](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386819873910807d8c322ca74d269c9f80f747330a52000)

### map()概念

<font color=red>map()</font>函数接收两个参数，一个是函数，一个是序列，map将传入的函数依次作用到序列的每个元素，并把结果作为新的列表返回。

##### 题目

> 利用<font color=red>map()</font>函数，把用户输入的不规范的英文名字，变为首字母大写，其他小写的规范名字。
> 例如输入：<font color=red>['adam', 'LISA', 'barT']</font>，输出：<font color=red>['Adam', 'Lisa', 'Bart']</font>

```Python
	>>> def test(name_list):
		    print(map(lambda name: name[0].upper()+name[1:].lower(), name_list))
		
	>>> test(['adam','LISA','barT'])
	['Adam', 'Lisa', 'Bart']
	>>> 
```

### reduce()概念

<font color=red>reduce()</font>把一个函数作用在一个序列[x1, x2, x3...]上，这个函数必须接收两个参数，reduce把结果继续和序列的下一个元素做累积计算

##### 题目

> Python提供的<font color=red>sum()</font>函数可以接受一个**list**并求和，请编写一个<font color=red>prod()</font>函数，可以接受一个list并利用<font color=red>reduce()</font>求积

```Python
	>>> def prod(num_list):
		    print reduce(lambda a,b : a*b , num_list)

	>>> prod([1,2,3,4])
	24
```

### filter概念

<font color=red>filter()</font>接收一个函数和一个序列，filter()把传入的函数依次作用于每个元素，然后根据返回值是True还是False决定保留还是丢弃该元素

##### 题目

> 请尝试用<font color=red>filter()</font>删除1~100的素数

概念补充： 素数，又称质数（prime number），有无限个。素数定义为在大于1的自然数中，除了1和它本身以外不再有其他因数的数称为质数

```Python
	#coding=utf-8
	
	#判断是不是素数，不是返回True，是返回False
	def not_prime(num):
	      if(num < 2):
	            return True
	      judge = 2
	      while(judge < num):
	            if num%judge == 0:
	                  return True
	            judge += 1
	      return False
	
	#将一个数字列表中所有的素数过滤删除掉
	def prime_number(num_list):
	      print filter(not_prime,num_list)
	
	#删除1~100以内的素数
	prime_number(range(100))
```

```Python
	运行结果：
	==================== RESTART: E:\Python\practices\test.py ====================
	[0, 1, 4, 6, 8, 9, 10, 12, 14, 15, 16, 18, 20, 21, 22, 24, 25, 26, 27, 28, 30, 32, 33, 34, 35, 36, 38, 39, 40, 42, 44, 45, 46, 48, 49, 50, 51, 52, 54, 55, 56, 57, 58, 60, 62, 63, 64, 65, 66, 68, 69, 70, 72, 74, 75, 76, 77, 78, 80, 81, 82, 84, 85, 86, 87, 88, 90, 91, 92, 93, 94, 95, 96, 98, 99]
```

