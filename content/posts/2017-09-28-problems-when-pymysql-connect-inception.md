---
title: 自动化DDL审核|pymysql链接Inception中踩过的几个坑
date: 2017-09-28 19:12:48
categories: [Mysql, Python]
tags: [pymsql, Inception, Python]
---

简单整理了一下这几天使用 pymysql 链接 inception 做 sql 审核过程中出现的坑和解决方法。

这些解决方法一定不是最漂亮的，如果有伙伴遇到同样的问题，并且有好的解决方案，请一定在留言中回一下我哦。

先行感激！！

<!--more-->

## Pymysql 在链接 inception 在判断版本时出现 value error

### 原因：

pymysql 通过`.`进行分割，但是 Inception 的版本信息是这样的

`Ver 14.14 Distrib Inception2.1.50, for Linux (x86_64) using EditLine wrapper`

正常 Mysql 的版本号是：

`mysql Ver 14.14 Distrib 5.7.18, for Linux (x86_64) using EditLine wrapper`

因此 Pymysql 获取到的版本值为 Inception2，所以在内部强制类型转换`int()`的时候报 value error。

### 解决方法（修改 pymysql 源码）：

修改 Pymysql 文件夹内的 connections.py

```python
    def _request_authentication(self):
        # https://dev.mysql.com/doc/internals/en/connection-phase-packets.html#packet-Protocol::HandshakeResponse
        if self.server_version.split('.', 1)[0] == 'Inception2':
            self.client_flag |= CLIENT.MULTI_RESULTS
        elif int(self.server_version.split('.', 1)[0]) >= 5:
            self.client_flag |= CLIENT.MULTI_RESULTS
```

这样的方式可以解决问题，但总觉得不妥，没法普及，不然需要带着修改过的 Pymysql 包 -。-|| )

## Inception 始终反馈"Must start as begin statement"的语法错误

> 注意！ 这个问题是出现在正确书写了 inception 的语法格式情况下，
>
> 语法规则:
>
> /_--user=username;--password=xxxx;--host=127.0.0.1;--port=3306;_/
>
> inception_magic_start;
>
> ... # 这里写 sql 语句；
>
> inception_maigc_commit;
>
> 如果是语法问题，请先自行检查语法。

### 原因：

pymysql 模块会自动向 inception 发送 SHOW WARNINGS 语句，而恰好这个 warnings 被 inception 捕捉到了，从而导致 inception 返回一个"Must start as begin statement"错误被 archer 捕捉到报在日志里.

### 解决方法：

很遗憾，目前的方法我依然是修改 pymsql 的源码，请修改`pymysql/cursors.py:338行`，将`self._show_warnings()`这一句注释掉，换成 pass，如下：

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/modify-pymysql.png)

依旧，这个方法有副作用，会导致所有调用该 pymysql 模块的程序不能 show warnings，

##### 这个问题的解决方法：

> 推荐使用 virtualenv 或 venv 创建一个独立的 python 运行环境来管理包

## 执行时，出现 Commands out of sync 的错误

### 原因：

这个原因出现在应用层的话，检查在`fetch all`前面是否有`commit`，去掉之后，问题就解决了。

不需要手动 commit,默认会自动 commit
