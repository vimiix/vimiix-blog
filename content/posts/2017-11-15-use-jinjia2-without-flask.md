---
title: '脱离Flask上下文，使用jinja2渲染html模板'
date: 2017-11-15 00:59:48
categories: 'jinja2'
tags: [Python, 'template', jinja2]
---

## 前言

首先，如果一个正常的 flask 带路由的接口，我们是不需要关心上下文对象的，Flask 做了很多“魔术”的方法，当一个 Flask 应用接收到一个请求的时候，它会在将逻辑委托给你的视图函数之前，创建好一个上下文对象。

当我们返回的时候调用`render_template(template, **context)`，就可以正常的渲染界面返回，在这个函数中，如果看一下源码就会发现，返回渲染之前，会创建一个 ctx 去获得当前环境的`app`变量。然后通过这个 ctx 去渲染传进来的`context`参数列表。

<!--more-->

```python
def render_template(template_name_or_list, **context):
    """Renders a template from the template folder with the given
    context.

    :param template_name_or_list: the name of the template to be
                                  rendered, or an iterable with template names
                                  the first one existing will be rendered
    :param context: the variables that should be available in the
                    context of the template.
    """
    ctx = _app_ctx_stack.top
    ctx.app.update_template_context(context)
    return _render(ctx.app.jinja_env.get_or_select_template(template_name_or_list),
                   context, ctx.app)
```

基于之前知道有这个函数可以替换 html 中的模板变量，今天在写代码的过程中，需要将一个 html 文件中变量动态更新后通过邮件发送走。出于惯性思维，理所当然的就直接撸代码：

```python
from flask import render_template
# 要更新模板中的用户名和密码,返回要邮件发送的内容
message = render_template("email.html", name="xxx", password='xxx')
```

一运行结果就报错：

```python
Traceback (most recent call last):
  File "xxx.py", line 14, in <module>
    message = render_template("email.html", name="xxx", password='xxx'))
  File "/usr/local/lib/python2.7/dist-packages/flask/templating.py", line 123, in render_template
    ctx.app.update_template_context(context)
AttributeError: 'NoneType' object has no attribute 'app'
```

经过一番摸索，大概知道这个`ctx`为嘛是个`None`，在`werkzeug/local.py`中有说明：

```python
@property
def top(self):
    """The topmost item on the stack.  If the stack is empty,
    `None` is returned.
    """
    try:
        return self._local.stack[-1]
    except (AttributeError, IndexError):
        return None
```

果不其然，在这段代码中，我直接调用了`render_template()`,并没有通过 Flask 路由，所以，上下文对象并没有被创建。而`render_template()`尝试去通过`ctx`获取当前应用的`app`对象，所以 ctx 被赋值为 None,因而出现上面的报错信息。

## 提出问题

这个问题该如何解决呢？我在这个地方，并不需要创建也不应该创建一个 flask 应用，但我又想使用 jinja2 的模板语言去更新数据。

## 解决问题

首先，发生这个错误，是因为 Flask 帮我们代理了 jinja2 的渲染动作，提供出一些更适合开发者使用的接口。也就是 Flask 把 jinja2 藏在了它内部，包装了一下，我们想直接用里面的东西，Flask 就不干了。明白人都看得出来，一切都是 Flask 引起的，负全责，jinja2 是清白的。既然这样，我们的解决方法就是拨调 flask 的包装， 直接去调用 jinja2 的渲染机制，这个过程就需要我们自己来写个小函数实现渲染功能。

示例代码如下：

```python
import jinja2

def render_without_request(template_name, **context):
    """
    用法同 flask.render_template:

    render_without_request('template.html', var1='foo', var2='bar')
    """
    env = jinja2.Environment(
        loader=jinja2.PackageLoader('package','templates')
    )
    template = env.get_template(template_name)
    return template.render(**context)
```

这个函数假设你的应用程序文件结构普通的那种。具体来说，你的项目结构应该是这个样子：

```
package/
├── __init__.py <--- init文件必须保留，python包的标志
└── templates/
    └── template.html
```

这里需要主要就是关注一下 jinja2 的`PackageLoader`这个函数，设计到文件之间的引用关系，可以参考一下源码,在`jinjia2/loaders.py`文件：

```python
class PackageLoader(BaseLoader):
    """Load templates from python eggs or packages.  It is constructed with
    the name of the python package and the path to the templates in that
    package::

        loader = PackageLoader('mypackage', 'views')

    If the package path is not given, ``'templates'`` is assumed.

    Per default the template encoding is ``'utf-8'`` which can be changed
    by setting the `encoding` parameter to something else.  Due to the nature
    of eggs it's only possible to reload templates if the package was loaded
    from the file system and not a zip file.
    """
```

初始化这个类的时候，第一个参数是模板所在的目录包名，第二个是模板目录名，可选，默认为`templates`，第三个可选参数是`encoding`,默认为`utf-8`

至此我们这个函数写好以后，就可以直接像`flask.render_template()`一样来调用了，它可以单纯的帮助我们把 html 模板渲染出来啦！

单纯多美好~~^.^
