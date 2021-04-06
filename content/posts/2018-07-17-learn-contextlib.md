---
title: '浅谈Python中的上下文管理'
date: 2018-07-18 00:56:48
categories: 'Python'
tags: ['Python', 'context']
---

## with 语法

平常在写 Python 代码的时候，经常会用到`with` 来处理一个上下文环境，比如文件的打开关闭，数据库的连接关闭等等。

`with`语法的使用，需要我们处理的对象实现`__enter__`和`__exit__`两个魔术方法来支持。`__enter__`函数处理逻辑函数之前需要做的事情，并返回操作对象作为`as`后面的变量，`__exit__`函数处理当代码离开`with`代码块以后的事情。

`with`语法非常方便的让我使用资源并且不用操心忘记后续操作所带来的隐患。

<!--more-->

下面是一个简单的自己实现支持`with`的类对象示例：

```python

class MyContextManager(object):

    def __enter__(self):
        print("Hello")
        return self

    def __exit__(self, *args):
        print("Bye")

    def work(self):
        print("Do something...")

with MyContextManager() as worker:
    worker.work()

```

运行以后结果为：

```python
Hello
Do something...
Bye
```

并且`with`语法还支持嵌套，可以同时打开多个上下文环境，有时候这对于我们同时操作多个对象是很方便的，例如我们需要同时打开两个文件，一个读一个写，这时候就可以这样写：

```python
with open(file_A) as reader, open(file_B, 'w') as writer:
    for index, line in enumerate(reader):
        writer.write(line)
```

## contextlib 内置库

除了我们熟悉的在对象中实现`__enter__`和`__exit__`方法来支持`with`语法外，python 还自带一个用于上下文管理处理的 [**contextlib**](https://github.com/python/cpython/blob/master/Lib/contextlib.py) 库。

contextlib 库是一个非常优美的库，它是为了加强`with`语句，提供上下文机制的模块，它是通过 Generator 实现的。通过定义类以及写`__enter__`和`__exit__`来进行上下文管理虽然不难，但是很繁琐。contextlib 中的 contextmanager 作为装饰器来提供一种针对函数级别的上下文管理机制。

时至今日，contextlib 库中不仅提供同步的`contextmanager`装饰器，同时也支持编写协程时处理异步的上下文环境的`asynccontextmanager`装饰器。两者的用法稍有不同，但思路是相同的。

###@contextmanager

对于`contextmanager`装饰器的原理，通过阅读源码，我的理解是：**"插入式编码"**。这里我有思考过和普通的装饰器怎么的不同，我自己定义为普通的装饰器是**"包裹式编码"**，看一个装饰器的功能往往是从装饰角度由外向内观察逻辑，而`contextmanager`却不同，它是"插入式"的，需要从函数出发由内向外观察。

怎么说是插入式呢？一个函数原本是自上向下顺序执行，突然在代码的中间，我们想要做点什么，就把代码卡在这里，去做想做的事情，等做完了以后，再回来接着执行相应的代码。

要做到这一点，就借助到了`yield`关键字，`contextmanager`接受一个 Generator 来借助`with ... as ..`的语法特性，在内部实现了`__enter__`和`__exit__`方法后，将`yield`返回的对象输出出去，这样就可以衔接上了。比较官方的用法是下面这样：

```python
@contextmanager
def some_generator():
    # 这里处理之前的事情
    try:
        yield value#这里 yield 返回要操作的变量
    finally:
        # 这里处理之后的事情

with some_generator() as variable:
    # 处理逻辑
```

这样的写法执行顺序就等价于：

```python
# 这里处理之前的事情
try:
    variable = value
    # 处理逻辑
finally:
    # 这里处理之后的事情
```

### @asynccontextmanager

`asynccontextmanager`装饰器和`contextmanager`类似，但其内部实现是通过`async`和`await`协程语法实现的，所以它装饰的函数也必须是异步的实现。（这个语法支持需要 python 版本大于 3.5）

示例逻辑代码如下：

```python
@asynccontextmanager
async def some_async_generator():
    # 这里处理之前的事情
    try:
        yield value#这里 yield 返回要操作的变量
    finally:
        # 这里处理之后的事情

async with some_async_generator() as variable:
     # 处理逻辑
```
