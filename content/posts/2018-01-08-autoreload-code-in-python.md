---
title: '[python]web框架中的代码自动重载怎么实现'
date: 2018-01-08 22:37:48
categories: 'Python'
tags: [Python, 'reload', Flask, Django, uWSGI]
---

在开发和调试 wsgi 应用程序时，有很多方法可以自动重新加载代码。例如，如果你使用的是`werkzeug`，则只需要传`use_reloader`参数即可：

```python
run_sumple('127.0.0.1', 5000, app, use_reloader=True)
```

对于 Flask,实际上在内部使用 werkzeug，所以你需要设置 debug = true：

```python
app.run(debug=True)
```

django 会在你修改任何代码的时候自动为你重新加载：

```python
python manage.py runserver
```

所有这些例子在本地开发的时候都非常有用，但是，建议不要在实际生产中使用。

作为学习，可以一起来看一下，python 是如何让代码自动地重新加载的？

<!--more-->

## uWSGI

如果使用的是 `uwsgi` 和 `django` ，实际上可以直接通过代码跳转查看一下 `django` 本身的自动重载机制：

```python
import uwsgi
from uwsgidecorators import timer
from django.utils import autoreload

@timer(3)
def change_code_gracefull_reload(sig):
    if autoreload.code_changed():
        uwsgi.reload()
```

可以看出，`django` 通过一个定时器在不断的监测代码是否有变化，以此来出发重新加载函数。

如果你使用的是其他框架，或者根本没有框架，那么可能就需要在应用程序中自己来实现代码改动监测。这里是一个很好的示例代码，借用和修改自 [cherrypy](http://cherrypy.org/)：

```python
import os, sys

_mtimes = {}
_win = (sys.platform == "win32")

_error_files = []

def code_changed():
    filenames = []
    for m in sys.modules.values():
        try:
            filenames.append(m.__file__)
        except AttributeError:
            pass
    for filename in filenames + _error_files:
        if not filename:
            continue
        if filename.endswith(".pyc") or filename.endswith(".pyo"):
            filename = filename[:-1]
        if filename.endswith("$py.class"):
            filename = filename[:-9] + ".py"
        if not os.path.exists(filename):
            continue # File might be in an egg, so it can't be reloaded.
        stat = os.stat(filename)
        mtime = stat.st_mtime
        if _win:
            mtime -= stat.st_ctime
        if filename not in _mtimes:
            _mtimes[filename] = mtime
            continue
        if mtime != _mtimes[filename]:
            _mtimes.clear()
            try:
                del _error_files[_error_files.index(filename)]
            except ValueError:
                pass
            return True
    return False
```

你可以将上面的内容保存在你的项目中的 `autoreload.py` 中，然后我们就可以像下面这样去调用它（类似于 `django` 的例子）：

```python
import uwsgi
from uwsgidecorators import timer
import autoreload

@timer(3)
def change_code_gracefull_reload(sig):
    if autoreload.code_changed():
        uwsgi.reload()
```

## gunicorn

对于 `gunicorn` ，我们需要写一个脚本来 `hook（钩子:触发）` 到 `gunicorn` 的配置中：

```python
import threading
import time
try:
    from django.utils import autoreload
except ImportError:
    import autoreload

def reloader(server):
    while True:
        if autoreload.code_changed():
            server.reload()
        time.sleep(3)

def when_ready(server):
    t = threading.Thread(target=reloader, args=(server, ))
    t.daemon = True
    t.start()
```

你需要把上面的代码保存到一个文件中，比如说 `config.py` ，然后像下面这样传给 `gunicorn` ：

```bash
gunicorn -c config.py application
```

## 外部解决方法

你也可以通过正在使用的 `wsgi` 服务系统本身以外的一些方法来实现重启系统，它只需发出一个信号，告诉系统重启代码，比如可以使用 [watchdog](https://github.com/gorakhargosh/watchdog)。例如：

```bash
watchmedo shell-command --patterns="*.py" --recursive --command='kill -HUP `cat /tmp/gunicorn.pid`' /path/to/project/
```

> 转载请保留以下信息：
>
> 原文链接： [http://www.vimiix.com/post/2018/01/08/autoreload-code-in-python/](http://www.vimiix.com/post/2018/01/08/autoreload-code-in-python/)
