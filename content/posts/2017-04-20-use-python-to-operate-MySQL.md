---
title: 'Python操作MySQL数据库'
date: 2017-04-20 23:57:34
tags:
  - Python
  - MySQL
categories: Python
---

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/python+mysql.jpg)

<!--more-->

## 数据库准备

在操作数据库之前，先做一些前提准备

进入数据库

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/sql_login.png)

创建一个名字叫做<font color=red>[test]</font>的数据库。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/sql_create.png)

在这个数据库中，创建一张表，暂且称之为<font color=red>[user]</font>，

这张表里面包含两个标签：<font color=red>[name]</font> 和 <font color=red>[password]</font>

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/sql_create_table.png)

给这张表里面插入两个数据

- Vimiix,123456
- Mike,654321

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/sql_insert.png)

到这里，就做好了一个为后面测试操作所用的数据库。

## 安装 MySQLdb

python 要操作 MySQL 数据库需要用到 MySQLdb 模块，所以需要先安装这个模块。

我的电脑是 win7 64 位系统，所以可以直接在网上下载 MySQLdb 的安装程序，运行即可。

> 保证 MySQLdb 模块的安装路径在 Python 的安装目录下 Lib/site-packages 就可以

MySQLdb 下载地址：

- 32 位：[http://download.csdn.NET/detail/seven_zhao/6607621](http://download.csdn.NET/detail/seven_zhao/6607621)
- 64 位：[http://download.csdn.Net/detail/seven_zhao/6607625](http://download.csdn.Net/detail/seven_zhao/6607625)

## Python 操作 MySQL 数据库

直接上带注释代码：

```Python
	#coding=utf-8

	import MySQLdb

	def sql_operation():
	    #打开数据库连接
	    db = MySQLdb.connect(host='localhost',user='root',passwd='123456',db='test')
	    #使用cursor()方法获取操作游标
	    cursor = db.cursor()
		#SQL语句：更新user表中Vimiix的password值为888888
	    sql = "update user set password='%s' where name='Vimiix'"%('888888',)
	    try:
	        #执行SQL语句
	        cursor.execute(sql)
	        #提交到数据库执行
	        db.commit()
	    except:
	        print('Update Error!')
	        #发生错误回滚
	        db.rollback()
	    finally:
			#关闭数据库
	        db.close()

	if __name__ == "__main__":
	    sql_operation()
```

通过 python 解释器执行上面的代码以后，回到数据库查看 user 表中的内容，此时表中 Vimiix 的 password 已经被修改为‘888888’了。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/sql_update.png)

## 补充一些常用的方法：

#### cursor 用来执行命令的方法:

**callproc(self, procname, args)**:用来执行存储过程,接收的参数为存储过程名和参数列表,返回值为受影响的行数

**execute(self, query, args)**:执行单条 sql 语句,接收的参数为 sql 语句本身和使用的参数列表,返回值为受影响的行数

**executemany(self, query, args)**:执行单挑 sql 语句,但是重复执行参数列表里的参数,返回值为受影响的行数

**nextset(self)**:移动到下一个结果集

#### cursor 用来接收返回值的方法:

**fetchall(self)**:接收全部的返回结果行.

**fetchmany(self, size=None)**:接收 size 条返回结果行.如果 size 的值大于返回的结果行的数量,则会返回 cursor.arraysize 条数据.

**fetchone(self)**:返回一条结果行.

**scroll(self, value, mode='relative')**:移动指针到某一行.如果 mode='relative',则表示从当前所在行移动 value 条,如果 mode='absolute',则表示从结果集的第一行移动 value 条.
