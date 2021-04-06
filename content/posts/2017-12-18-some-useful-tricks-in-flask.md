---
title: '写Flask应用时的一些优雅技巧'
date: 2017-12-18 18:56:48
categories: 'flask'
tags: [python, flask, pythonic, tricks]
---

## 借助`find_modules`,`import_string`优雅地注册蓝图模块

[`find_modules`](http://werkzeug.pocoo.org/docs/0.13/utils/#werkzeug.utils.find_modules), [`import_string`](http://werkzeug.pocoo.org/docs/0.13/utils/#werkzeug.utils.import_string)这两个函数包含在`werkzeug.utils`工具包中，借助着两个工具函数可以帮助我们在更优雅的给应用注册`blueprint`模块，尤其是当项目中`blueprint`模块很多的时候，会节省很多行代码，看起来更加的舒服。

<!--more-->

#### import_string(import_name, silent=False)

import_string 可以通过字符串导出需要导入的模块或对象：

**参数**

- import_name：要导入的对象的模块或对象名称
- silent：如果设置为 True，则忽略导入错误，相反则返回 None

#### find_modules(import_path, include_packages=False, recursive=False)

找到一个包下面的所有模块,这对于自动导入所有蓝图模块是非常有用的

**参数**

- import_path：包路径
- include_packages：如果设置为 True,会返回包内的子包
- recursive：是否递归搜索子包

#### 示例代码

_blueprints/example.py_

```python
# 模块部分
# create  blueprint :)
bp = Blueprint('bp_name', __name__)

```

_app.py_

```python
# 注册部分
def register_blueprints(app):
    """Register all blueprint modules"""
    for name in find_modules('blueprints'):
        module = import_string(name)
        if hasattr(module, 'bp'):
            app.register_blueprint(module.bp)
    return None
```

## 使用 Flask 中的 flash 闪存传递反馈信息

flask 的闪存系统主要是用来想用户提供反馈信息。内容一般是对用户上一次请求中的操作给出反馈。反馈信息存储在服务端，用户可以在本次（且只能在本次）请求中访问上一次的反馈信息，当用户获得了这些反馈信息以后，就会被服务端删除。Flask 为 jinja2 开放了一个[`get_flashed_messages(with_categories=False, category_filter=[])`](http://flask.pocoo.org/docs/0.12/api/#flask.get_flashed_messages)函数来获取上一次的闪存信息,这个函数可以**直接在模板中使用**。

**参数**

- with_categories：True 返回元祖，False 返回消息本身
- category_filter：过滤分类关键词（字符串或列表）

后台当请求结束准备返回的时候，使用[`flash(message, category='message')`](http://flask.pocoo.org/docs/0.12/api/#flask.flash)函数来为下次请求保存一条反馈信息。

**参数**

- message：信息文本
- category：自定义分类关键词

#### [官方示例代码](http://flask.pocoo.org/docs/0.12/patterns/flashing/#message-flashing-pattern)

## 使用 Flask 中内置日志系统发送错误日志邮件

Flask 使用 python 内置的日志系统，它实际上可以发送错误邮件。

#### 示例代码：

```python
ADMINS = ['yourname@example.com']
if not app.debug:
    import logging
    from logging.handlers import SMTPHandler
    mail_handler = SMTPHandler('127.0.0.1', #邮件服务器
                               'server-error@example.com', #发件人
                               ADMINS, #收件人
                               'YourApplication Failed') #邮件主题
    mail_handler.setLevel(logging.ERROR)
    app.logger.addHandler(mail_handler)
```

还可以更进一步，将错误日志格式化，方便阅读：

```python
from logging import Formatter
mail_handler.setFormatter(Formatter('''
Message type:       %(levelname)s
Location:           %(pathname)s:%(lineno)d
Module:             %(module)s
Function:           %(funcName)s
Time:               %(asctime)s

Message:

%(message)s
'''))
```

关于 SMTPHandler 的介绍，访问[官网 SMTPHandler 手册](https://docs.python.org/dev/library/logging.handlers.html#smtphandler)

## 提前中断请求返回错误码，并定制相应错误页面

在 Flask 中我们能够用`redirect()`函数重定向用户到其它地方。还能够用 [`abort()`](http://flask.pocoo.org/docs/0.12/api/#flask.abort) 函数提前中断一个请求并带有一个错误代码。

#### 示例代码

```python
from flask import abort, redirect, url_for

@app.route('/')
def index():
    return redirect(url_for('login'))

@app.route('/login')
def login():
    abort(404)
    this_is_never_executed() # 永远不会被执行到
```

配合 Flask 提供的 [`errorhandler()`](http://flask.pocoo.org/docs/0.12/patterns/errorpages/#error-handlers) 装饰器定制自己的相应错误界面

```python
from flask import render_template

@app.errorhandler(404)
def page_not_found(error):
    return render_template('page_not_found.html'), 404
```

注意到 `404` 是在 `render_template()` 调用之后。告诉 Flask 该页的错误代码应是 `404` ， 即没有找到。``

> 原文链接：[http://vimiix.com/post/2017/12/18/some-useful-tricks-in-flask/](http://vimiix.com/post/2017/12/18/some-useful-tricks-in-flask/)
