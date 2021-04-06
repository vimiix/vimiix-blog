---
title: '[python]ImportError:attempted relative import with no known parent package'
date: 2017-12-29 12:37:48
categories: 'Python'
tags: [Python, '翻译', importError]
---

## 前言

在这篇文章中，我将会解析 `ImportError: attempted relative import with no known parent package` 这个异常的原因。当你在运行的 python 脚本。使用了相对引用方式 _(类似`import ..module`)_ 去引用包时，可能会出现这个异常。

<!--more-->

让我们来看看发生这个异常的例子。

## 问题

假设你有以下目录结构：

```shell
project
├── config.py
└── package
    ├── __init__.py
    └── demo.py
```

`config.py` 中包含一些应该在 `demo.py` 中使用的变量

- project/config.py

```python
count = 5
```

- project/package/demo.py

```python
from .. import config
print("The value of config.count is {0}".format(config.count))
```

当我们尝试运行`demo.py`时，会遇到以下错误：

```python
E:\project> python demos/demo.py
Traceback (most recent call last):
  File "demos/demo.py", line 1, in <module>
    from .. import config
ImportError: attempted relative import with no known parent package
```

python 解释器抛出了没有父级包的异常。为什么？

让我们看看 python 解释器是如何解析相关模块。从 [PEP 328](https://www.python.org/dev/peps/pep-0328/) 中，我们找到了关于 `the relative imports`（相对引用）的介绍：

> Relative imports use a module’s **\_\_name\_\_** attribute to determine that module’s position in the package hierarchy. If the module’s name does not contain any package information (e.g. it is set to **\_\_main\_\_** ) then relative imports are resolved as if the module were a top level module, regardless of where the module is actually located on the file system.
>
> 相对导入通过使用模块的 **\_\_name\_\_** 属性来确定模块在包层次结构中的位置。如果该模块的名称不包含任何包信息（例如，它被设置为 **\_\_main\_\_** ），那么相对引用会认为这个模块就是顶级模块，而不管模块在文件系统上的实际位置。

换句话说，解决模块的算法是基于`__name__`和`__package__`变量的值。大部分时候，这些变量不包含任何包信息 ---- 比如：当 `__name__` = `__main__` 和 `__package__` = `None` 时，python 解释器不知道模块所属的包。在这种情况下，相对引用会认为这个模块就是顶级模块，而不管模块在文件系统上的实际位置。

为了演示这个原理，我们来更新一下代码：

- project/config.py

```python
print('__file__={0:<35} | __name__={1:<20} | __package__={2:<20}'.format(__file__,__name__,str(__package__)))
count = 5
```

- project/package/demo.py

```python
print('__file__={0:<35} | __name__={1:<20} | __package__={2:<20}'.format(__file__,__name__,str(__package__)))
from .. import config
print("The value of config.count is {0}".format(config.count))
```

再次尝试运行一下，会得到以下输出：

```python
E:\project> python demos/demo.py
__file__=demos/demo.py      | __name__=__main__    | __package__=None
Traceback (most recent call last):
  File "demos/demo.py", line 3, in <module>
    from .. import config
ImportError: attempted relative import with no known parent package
```

正如我们所看到的，python 解释器没有关于模块所属的包的任何信息（ `__name__` = `__main__` 和 `__package__` = `None` ），因此它抛出了找不到父级包的异常。

## 解决方案一

- 我们通过在其中创建一个新的空 `__init__.py` 文件来将项目目录转换为一个包。

- 我们在项目目录的父目录中创建一个文件 `main.py`

```shell
toplevel
├── main.py
└── project
  ├── __init__.py
  ├── config.py
  └── package
      ├── __init__.py
      └── demo.py
```

- toplevel/main.py

```python
print('__file__={0:<35} | __name__={1:<20} | __package__={2:<20}'.format(__file__,__name__,str(__package__)))
import project.demos.demo
```

执行一下新的示例，输出如下：

```python
E:\toplevel>python main.py
__file__=main.py                             | __name__=__main__             | __package__=None
__file__=E:\toplevel\project\demos\demo.py   | __name__=project.demos.demo   | __package__=project.demos
__file__=E:\toplevel\project\config.py       | __name__=project.config       | __package__=project
The value of config.count is 5
```

在 `main.py` 中导入 `project.demos.demo` 会设置相对引用的包信息（ `__name__` 和 `__package__` 变量）。现在，python 解释器可以成功解析 `project\demos\demo.py` 中的相对引用了。

## 解决方案二

- 我们通过在 `project` 文件夹中创建一个新的空 `__init__.py` 来将 `project` 目录转换为一个包。

- 在 `toplevel` 目录下通过 `-m` 参数来调用 python 解释器，去执行 `project.demos.demo`<sup>[\[1\]](#note1)</sup>

```shell
toplevel
└── project
  ├── __init__.py
  ├── config.py
  └── package
      ├── __init__.py
      └── demo.py
```

再次执行：

```python
E:\toplevel>python -m project.demos.demo
__file__=E:\toplevel\project\demos\demo.py   | __name__=__main__        | __package__=project.demos
__file__=E:\toplevel\project\config.py       | __name__=project.config  | __package__=project
The value of config.count is 5
```

运行该命令将自动设置包信息（`__package__`变量）。现在，python 解释器可以成功解析 `project\ demos\demo.py` 中的相对引用了（甚至认为 `__name __` = `__ main__` ）。

[1] <a name="note1"></a>**译者注：注意使用 `-m` 参数的时候，后面指定的执行文件没有 `.py` 后缀**

import-error-relative-no-parent

> **注：转载请保留下面的内容**
>
> 原文链接：[https://www.napuzba.com/story/import-error-relative-no-parent/](https://www.napuzba.com/story/import-error-relative-no-parent/)
>
> 译文链接：[http://vimiix.com/post/2017/12/29/import-error-relative-no-parent/](http://vimiix.com/post/2017/12/29/import-error-relative-no-parent/)
