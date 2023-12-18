---
title: 'macOXS中使用matplotlib遇到的问题及探究'
date: 2018-10-16 11:22:48
categories: 'Python'
tags: ['Python', 'note', 'matplotlib', 'solution']
---

第一次在 mac 系统上使用**matplotlib**库的时候，在执行的时候，往往会遇到下面这样的问题：

```bash
ImportError: Python is not installed as a framework. balabala....
```

### 解决方案

当然这个问题很好解决，网上有一搜就会找到如下两种解决方案：

#### 第一种方案是在系统中设置

- 假设你已经通过`pip install matplotlib`安装了 matplotlib，那么在你的根目录中会有一个名为`〜/ .matplotlib`的目录。
- 在这个目录中创建一个`matplotlibrc`的文件 ,在里面添加一行代码：`backend: TkAgg`，保存退出即可。

总结为一行 shell 命令就是：`echo "backend: TkAgg" >> ~/.matplotlib/matplotlibrc`

这种方式可以设定整个系统的 matplotlib 渲染使用的引擎，但是不好的是，代码会变得不可移植，如果服务器很多，我们需要每一台机器都去设置这个参数，这时候就需要使用第二种方案。

<!--more-->

#### 第二种方案是在代码中设置

在引用`matplotlib`库的代码之前，添加如下两行代码（确认安装 TkInter）：

```python
import matplotlib
matplotlib.use('TkAgg')
```

这样也可以临时的修改前面提到的 `backend`变量。

### 为什么要这样做？

从两种解决方法基本可以看出，两种方法都是通过修改一个叫做 `backend`的值来解决的。但这个`TKAgg`是什么鬼呢？

回答这个问题就需要知道为什么会默认的情况会产生上面的那个 `ImportError`。出现这个错误的原因是，在 mac os 系统镜像中渲染 matplotlib 的后端，默认情况下使用的是**Cocoa**的 API 来渲染。其实还有 **Qt4Agg** ， **GTKAgg** ， **TkAgg**等很多可以使用。这是 macosx 与其他 windows 或 linux os 相比不同的地方。

### 什么是`backend`？

网上有很多的文档或解决方案都会提到`backend`，但许多用户对此术语会感到困惑（比如我）。matplotlib 会面对许多不同的用法和输出形式。比如，有人会在 python shell 交互使用 matplotlib，并在键入命令时弹出绘图窗口。有人将 matplotlib 嵌入到图形用户界面（_如 wxpython 或 pygtk_）中以构建丰富的应用程序。还有人在批处理脚本中使用 matplotlib 从一些数值模拟生成[postscript 图像](https://zh.wikipedia.org/wiki/PostScript)，还有一些是在 Web 应用程序服务器中生成 posts 动画以提供动态图形。

为了支持所有这些用法，matplotlib 可以提供不同的输出，这些输出中都称为`backend`; `“frontend”`是面向用户的代码，即绘图代码，而`backend`完成幕后的所有生成图形的复杂工作。

matplotlib 中有两种类型的`backend`：

- 用户界面后端（用于 pygtk，wxpython，tkinter，qt4 或 macosx 等，也被称为“交互式后端”）
- 硬拷贝后端（用于制作图像文件 PNG，SVG，PDF，PS 等，也被称为“非交互式后端”）

设置`backend`的方法其实不止有网上提到两种方案，一共有四种方法。如果它们彼此有冲突，将使用以下列表中最后提到的方法，例如，调用`use()`则将会覆盖**matplotlibrc**中的设置。

1. **matplotlibrc**文件中的`backend`参数，也就是我们上面说的方案一：

   （如果想定制化 matplotlib 其他的参数，[点击查看](https://matplotlib.org/users/customizing.html#customizing-matplotlib)）：

```
backend : TkAgg   # Agg(antigrain) 渲染到 Tk canvas (需要安装TkInter)
```

2. 为当前 shell 或单个脚本设置`MPLBACKEND`环境变量：

```bash
> export MPLBACKEND="module://my_backend"
> python simple_plot.py

> MPLBACKEND="module://my_backend" python simple_plot.py
```

设置此环境变量将覆盖 matplotlibrc 中的 backend 参数值，即使当前工作目录中存在 matplotlibrc 也是如此。因此，**不鼓励**在全局设置`MPLBACKEND`，例如在`.bashrc`或`.profile`中，因为它会让我们很难发现问题。

3. 要为单个脚本设置 backend，也可以使用 `-d` 命令行参数：

```
> python script.py -dbackend
```

不推荐使用此方法，因为`-d`参数可能与解析命令行参数的脚本冲突（参考[issue＃1986](https://github.com/matplotlib/matplotlib/issues/1986)）。

4. 如果你的脚本依赖于特定的后端，则可以使用`use()`函数：

```python
import matplotlib
matplotlib.use('PS')   # 默认输出 postscript
```

如果要使用`use()`函数，则必须在导入`matplotlib.pyplot`之前调用。导入 pyplot 后调用`use()`将不起作用。如果想要使用不同的后端，则不得不对源码做编辑，修改`use()`中的代码来完成。因此，除非必要，否则应该避免显式调用`use()`。

> `backend`名称规范不区分大小写;例如，'GTKAgg'和'gtkagg'是等价的。

### `backend`选择列表

#### 交互式列表

这些是支持的用户界面和渲染器组合，能够显示到屏幕并使用非交互式列表中的适当渲染器写入文件：

| Backend   | Description                                                                                                                                                                                                                                                                                         |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GTKAgg    | Agg rendering to a [GTK](https://matplotlib.org/glossary/index.html#term-gtk) 2.x canvas (requires [PyGTK](http://www.pygtk.org/) and [pycairo](https://www.cairographics.org/pycairo/) or [cairocffi](https://pythonhosted.org/cairocffi/); Python2 only)                                          |
| GTK3Agg   | Agg rendering to a [GTK](https://matplotlib.org/glossary/index.html#term-gtk) 3.x canvas (requires [PyGObject](https://wiki.gnome.org/action/show/Projects/PyGObject) and [pycairo](https://www.cairographics.org/pycairo/) or [cairocffi](https://pythonhosted.org/cairocffi/))                    |
| GTK       | GDK rendering to a [GTK](https://matplotlib.org/glossary/index.html#term-gtk) 2.x canvas (not recommended and d eprecated in 2.0) (requires [PyGTK](http://www.pygtk.org/) and [pycairo](https://www.cairographics.org/pycairo/) or [cairocffi](https://pythonhosted.org/cairocffi/); Python2 only) |
| GTKCairo  | Cairo rendering to a [GTK](https://matplotlib.org/glossary/index.html#term-gtk) 2.x canvas (requires [PyGTK](http://www.pygtk.org/) and [pycairo](https://www.cairographics.org/pycairo/) or [cairocffi](https://pythonhosted.org/cairocffi/); Python2 only)                                        |
| GTK3Cairo | Cairo rendering to a [GTK](https://matplotlib.org/glossary/index.html#term-gtk) 3.x canvas (requires [PyGObject](https://wiki.gnome.org/action/show/Projects/PyGObject) and [pycairo](https://www.cairographics.org/pycairo/) or [cairocffi](https://pythonhosted.org/cairocffi/))                  |
| WXAgg     | Agg rendering to to a [wxWidgets](https://matplotlib.org/glossary/index.html#term-wxwidgets) canvas (requires [wxPython](https://www.wxpython.org/))                                                                                                                                                |
| WX        | Native [wxWidgets](https://matplotlib.org/glossary/index.html#term-wxwidgets) drawing to a [wxWidgets](https://matplotlib.org/glossary/index.html#term-wxwidgets) Canvas (not recommended and deprecated in 2.0) (requires [wxPython](https://www.wxpython.org/))                                   |
| TkAgg     | Agg rendering to a [Tk](https://matplotlib.org/glossary/index.html#term-tk) canvas (requires [TkInter](https://wiki.python.org/moin/TkInter))                                                                                                                                                       |
| Qt4Agg    | Agg rendering to a [Qt4](https://matplotlib.org/glossary/index.html#term-qt4) canvas (requires [PyQt4](https://riverbankcomputing.com/software/pyqt/intro) or `pyside`)                                                                                                                             |
| Qt5Agg    | Agg rendering in a [Qt5](https://matplotlib.org/glossary/index.html#term-qt5) canvas (requires [PyQt5](https://riverbankcomputing.com/software/pyqt/intro))                                                                                                                                         |
| macosx    | Cocoa rendering in OSX windows (presently lacks blocking show() behavior when matplotlib is in non-interactive mode)                                                                                                                                                                                |

#### 非交互式列表

以下是 matplotlib 渲染器的列表（每个渲染器都有一个同名支持，能够写入文件）：

| Renderer                                                       | Filetypes                                                                                                                                                                                                                                     | Description                                                                                                                                                             |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [AGG](https://matplotlib.org/glossary/index.html#term-agg)     | [png](https://matplotlib.org/glossary/index.html#term-png)                                                                                                                                                                                    | [raster graphics](https://matplotlib.org/glossary/index.html#term-raster-graphics) – high quality images using the [Anti-Grain Geometry](http://antigrain.com/) engine  |
| PS                                                             | [ps](https://matplotlib.org/glossary/index.html#term-ps) [eps](https://matplotlib.org/glossary/index.html#term-eps)                                                                                                                           | [vector graphics](https://matplotlib.org/glossary/index.html#term-vector-graphics) – [Postscript](https://en.wikipedia.org/wiki/PostScript) output                      |
| PDF                                                            | [pdf](https://matplotlib.org/glossary/index.html#term-pdf)                                                                                                                                                                                    | [vector graphics](https://matplotlib.org/glossary/index.html#term-vector-graphics) – [Portable Document Format](https://en.wikipedia.org/wiki/Portable_Document_Format) |
| SVG                                                            | [svg](https://matplotlib.org/glossary/index.html#term-svg)                                                                                                                                                                                    | [vector graphics](https://matplotlib.org/glossary/index.html#term-vector-graphics) – [Scalable Vector Graphics](https://en.wikipedia.org/wiki/Scalable_Vector_Graphics) |
| [Cairo](https://matplotlib.org/glossary/index.html#term-cairo) | [png](https://matplotlib.org/glossary/index.html#term-png) [ps](https://matplotlib.org/glossary/index.html#term-ps) [pdf](https://matplotlib.org/glossary/index.html#term-pdf) [svg](https://matplotlib.org/glossary/index.html#term-svg) ... | [vector graphics](https://matplotlib.org/glossary/index.html#term-vector-graphics) – [Cairo graphics](<https://en.wikipedia.org/wiki/Cairo_(graphics)>)                 |
| [GDK](https://matplotlib.org/glossary/index.html#term-gdk)     | [png](https://matplotlib.org/glossary/index.html#term-png) [jpg](https://matplotlib.org/glossary/index.html#term-jpg) [tiff](https://matplotlib.org/glossary/index.html#term-tiff) ...                                                        | [raster graphics](https://matplotlib.org/glossary/index.html#term-raster-graphics) – the [Gimp Drawing Kit](https://en.wikipedia.org/wiki/GDK) Deprecated in 2.0        |

### 参考链接

- <https://stackoverflow.com/questions/21784641/installation-issue-with-matplotlib-python>
- <https://matplotlib.org/faq/usage_faq.html#what-is-a-backend>
- <https://matplotlib.org/users/customizing.html#customizing-matplotlib>

--- EOF ---
