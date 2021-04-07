---
title: Python-运算符和流程控制
date: 2017-04-01 14:34:20
tags:
  - flow control
  - operators
  - Python
categories: Python

---

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/operator-flow-control.jpg)

随手笔记，本篇介绍了python中的各种运算符以及2种流程控制语句。

<!--more-->

## Python中的运算符

### 算术运算符

|运算符|描述|
|:-----:|:-----:|
|+|加|
|-|减|
|*|乘|
|/|除-Py2中9/2输出4,Py3中9/2输出4.5|
|%|取余（或者叫取模），返回除法的余数|
|**|幂-x**y，x的y次幂|
|//|取整（地板除）-返回商的整数部分，<br><font color=red>注意负数的取整，向左取整</font>|

### 比较（关系）运算符，运算结果为布尔值

|运算符|描述|
|:----:|:-----:|
|==|等于-比较对象是否相等|
|!=|不等于|
|>|大于|
|<|小于|
|>=|大于等于|
|<=|小于等于|

### 赋值运算符

|运算符|描述|
|:----:|:-----:|
|=|赋值|
|+=|a+=b 等效于 a=a+b|
|-=|a-=b 等效于 a=a-b|
|*=|a*=b 等效于 a=a*b|
|/=|a/=b 等效于 a=a/b|
|%=|a%=b 等效于 a=a%b|
|&#42;&#42;=|a&#42;&#42;=b 等效于 a=a&#42;&#42;b|
|//=|a//=b 等效于 a=a//b|

### 逻辑运算符

|运算符|描述|
|:----:|:-----:|
|and|逻辑与（同真为真，输出后值，有假即假，输出第一个假值）|
|or|逻辑或（同假为假，输出前值，有真即真，输出第一个真值）|
|not|逻辑非，取反|

### 位运算符

|运算符|描述|
|:----:|:-----:|
|&|按位与|

说明：参与预算的两个值，转换为二进制，如果两个相应位都为1，则该位结果为1，否则为0

栗子：计算 60 & 13


|十进制|二进制|十进制结果|
|:----:|:-----:|:----:|
|60|111100||
|13|001101||
||001100|12|

|运算符|描述|
|:----:|:-----:|
|&#124;|按位或|

说明：转换为二进制，只要对应的两个位有一个1，则该位结果为1，否则为0

栗子：计算 60 | 13

|十进制|二进制|十进制结果|
|:----:|:-----:|:----:|
|60|111100||
|13|001101||
||111101|61|

|运算符|描述|
|:----:|:-----:|
|^|按位异或：位相异结果为1|
|~|按位取反|
|<<|按位左移|
|>>|按位右移|


### 成员运算符

|运算符|描述|
|:----:|:-----:|
|in|如果在in后面的对象中找的到in前面的对象返回True,否则返回False|
|not in|找不到该成员返回True，否则返回False|

### 身份运算符

|运算符|描述|
|:----:|:-----:|
|is|is是判断两个标识符是不是引用自同一个对象（<font color=blue>理解：是否标记在同一块内存地址</font>）|
|is not|和is相反|

##### python内存管理机制，对于创建变量的时候，是否重新malloc内存的临界点为 -5~256 看下面的栗子：

```Python
	>>> a=256
	>>> b=256
	>>> a is b
	True
	>>> a=257
	>>> b=257
	>>> a is b
	False
	>>> a=-5
	>>> b=-5
	>>> a is b
	True
	>>> a = -6
	>>> b= -6
	>>> a is b
	False
```

<hr>

## 流程控制

### 理解

if判断语句：什么条件下干什么事

for,while循环语句:在什么条件下，一直做某事

### if流程语句

##### 语法：

```Python
	if 条件语句：
		执行代码1
	else:
		执行代码2
```

条件语句：必须是一个可以用bool值判断的

##### 流程图：

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/if-else.png)

##### 用户登录实例：

```Python
	#coding=utf-8
	username = 'admin' #设置默认用户名
	password = '123' #设置默认密码
	user_input = raw_input('Please input your name:') #提示用户输入用户名
	passwd_input = raw_input('Please input your password:') #提示用户输入密码
	if user_input == username and passwd_input ==password: #验证用户名和密码是否和默认值相同
		print 'welcome %s login.'%user_input
	else:
		print 'your name or password is wrong!'
```
##### 改进：

现在需要实现访客输入‘guest’后不需要输入密码，直接就可以登录进去；正常输入用户名的情况下还需要输入密码才能输入

```Python
	#coding=utf-8
	username = 'admin' #设置默认用户名
	password = '123' #设置默认密码
	guest_name = 'guest'
	user_input = raw_input('Please input your name:') #提示用户输入用户名
	if user_input == guest_name:
	      print 'welcome %s login.'%guest_name
	elif user_input == username:
	      passwd_input = raw_input('Please input your password:') #提示用户输入密码
	      if passwd_input ==password: #验证用户名和密码是否和默认值相同
	            print 'welcome %s login.'%user_input
	else:
	      print 'your name or password is wrong!'
```

运行结果：
![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/login.png)

### while循环

##### 语法：

```Python
	while 判断条件：
		循环语句
```

##### 实例1：计算1到100的和

```Python
	#coding=utf-8
	i = 1
	sum = 0
	while i<=100:
		sum += i
		i+=1
	print sum
```

##### 实例2：打印九九乘法表（while实现）

```Python
	#coding=utf-8
	i = 1
	j = 1
	while i < 10:
	      while j <= i:
	            if( j == i):
	                  print i,'x',j,'=',i*j
	            else:
	                  print i,'x',j,'=',i*j,' ',
	            j += 1
	      i += 1
	      j = 1
```

运行结果：

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/9x9.png)
