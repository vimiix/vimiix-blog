---
title: 'Python中的弱引用'
date: 2024-08-12 8:35:48
categories: 'python'
tags:
  - 'python'
  - 'weakref'
  - 'notes'
---

## 前言

在理解弱引用之前，我们需要先了解 python 中的对象引用和垃圾回收的基本概念，这样更有助于对弱引用的理解。

### 对象引用

在 Python 中，一切皆为对象，包括变量、函数、类以及其实例化的对象。我们要是有对象就必须通过赋值语句来创建一个引用（可以理解为标签），比如 `a = 1`，这里 `a` 就是对象 `1` 的引用。每个引用的所指向的对象都可以通过 `id()` 函数来查看，两个引用是否相同可以通过 `is` 运算符来比较。

*(`==`和 `is` 的区别：`==` 运算符比较对象的值（对象中保存的数据），而 `is` 比较引用的对象是否为同一个。)*

如果想要查看一个对象的内引用次数，可以通过 `sys.getrefcount(a) -1` 查看（*减1是因为调用 sys.getrefcount 函数传递参数时会增加一次引用*）。

**为了说明后面的弱引用，我们将计数的引用成为强引用。**

<!--more-->

## 垃圾回收

Python 中的对象在以下两种情况下会被视为垃圾，进而将其所占用的内存回收（GC）：

1. **没有任何引用指向该对象**；
2. 存在循环引用；（*循环引用解释器会判定为无法获取，所以都会被销毁，不在本文展开*）

`del` 语法可以实现删除一个对象的引用，而不是对象本身，当我们将指向目标对象的所有引用都删除后，才会触发 Python 的解释器的垃圾回收机制，回收该对象的内存地址。（**理解到这里就够，关于更详细的分代垃圾回收的算法等，请自行学习**）

另外，除了使用 `del` 来删除引用外，重新绑定也可能会导致对象引用数量归零触发垃圾回收。比如：

```python
a = [1]
a = [2]  # 此时 [1] 这个对象就会被销毁，因为引用 a 被指向了 [2] 这个对象，[1] 的引用数就归零了
```

## weakref 模块的原理

在 Python 中，内存管理是一个重要的话题。通过前言的铺垫，我们大概了解了 Python 是通过引用计数的方式实现内存管理机制来管理对象的生命周期的。然而，有些情况下，我们可能希望在不影响对象生命周期的前提下，引用某些对象，这时候就可以使用 Python 中的 `weakref` 模块。

`weakref`（弱引用）是一个用于跟踪对象的模块，它允许创建不会阻止对象被垃圾回收的引用。通常情况下，Python 中的强引用会增加对象的引用计数，当强引用计数变为零时，Python的垃圾回收器会自动回收该对象。然而，使用 `weakref` 创建的弱引用不会增加对象的引用计数，因此，当该对象的强引用计数降为零时，即使弱引用仍然存在，垃圾回收器也会回收该对象。

弱引用的存在使得我们可以创建对象的缓存、代理对象等场景，并且不会因不必要的引用而导致内存泄漏或对象无法及时回收。

## weakref 的使用方法

### 创建弱引用

在 Python 中，`weakref.ref` 函数可以用于创建弱引用。例如：

```python
import weakref

class A:
    def __init__(self, value):
        self.value = value

obj = A(10)
weak_ref = weakref.ref(obj)
print(weak_ref)  # 输出：<weakref at 0x...; to 'A' at 0x...>

# 通过弱引用访问原对象
print(weak_ref())  # 输出：<__main__.A object at 0x...>
```

在上述代码中，我们创建了一个 `A` 对象并将其传递给 `weakref.ref` 以创建一个弱引用。通过 `weak_ref()` 可以访问到原始对象。需要注意的是，如果原对象被垃圾回收，`weak_ref()`将返回 `None`。

### 弱引用的回调

在某些情况下，我们可能希望在弱引用所指向的对象被回收时执行特定的操作。这时，我们可以为弱引用注册一个回调函数。例如：

```python
import weakref

class A:
    def __del__(self):
        # 这里演示调用顺序，不建议自己在 __del__ 方法中自定义代码
        print("A object is being destroyed")

def on_finalize(reference):
    print(f"Object {reference} is being finalized.")

obj = A()
weak_ref = weakref.ref(obj, on_finalize)

# 删除强引用，触发垃圾回收
del obj
# 输出：A object is being destroyed
# 输出：Object <weakref at 0x...; dead> is being finalized.
```

在这里，我们为 `weakref.ref` 传递了一个回调函数 `on_finalize`。当对象被垃圾回收时，该回调函数会被触发。

### weakref.WeakValueDictionary

`weakref.WeakValueDictionary` 是 `weakref` 模块中常用的一个类，它提供了一个类似于字典的对象，其中的值是通过弱引用进行存储的。使用 `WeakValueDictionary` 可以避免缓存中的对象被意外持久化，从而有效地防止内存泄漏。

```python
import weakref

class A:
    def __init__(self, value):
        self.value = value

weak_dict = weakref.WeakValueDictionary()
obj1 = A(10)
obj2 = A(20)

weak_dict['a'] = obj1
weak_dict['b'] = obj2

print(weak_dict['a'])  # 输出：<__main__.A object at 0x...>

del obj1
print(weak_dict('a'))  # 输出：None
```

当 `obj1` 被删除时，对应的弱引用也会失效，因此在字典中查询 `'a'` 的值会返回`None`。

### weakref.WeakKeyDictionary

`weakref.WeakKeyDictionary` 与 `WeakValueDictionary` 类似，只不过它使用弱引用来存储字典的键，而不是值。这在需要根据对象特性进行缓存或者存储临时数据的场景中非常有用。

```python
import weakref

class A:
    def __init__(self, value):
        self.value = value

obj1 = A(10)
obj2 = A(20)

weak_dict = weakref.WeakKeyDictionary()
weak_dict[obj1] = 'First'
weak_dict[obj2] = 'Second'

print(weak_dict[obj1])  # 输出：First

del obj1
print(list(weak_dict.keys()))  # 仅输出 [<__main__.A object at 0x...>]，obj1 对应的键已删除
```

### weakref.ref() 和 weakref.proxy() 的区别

`weakref.ref()` 和 `weakref.proxy()` 都是用于创建对象的弱引用的工具，但它们在行为和应用场景上存在一些区别。

访问方式：

- `weakref.ref()` 返回的是一个弱引用对象，你需要调用这个对象才能访问原始对象，并且需要手动处理可能返回None的情况。
- `weakref.proxy()` 返回的是一个代理对象，直接使用它就可以像使用原始对象一样操作，但如果原始对象被回收，访问时会抛出异常。

举个实例便于理解：

```python
import weakref
class A:
    def __init__(self, value):
            self.value = value

ref = weakref.ref(c_obj)
ref() # <__main__.C object at 0x...>
ref().value # 1

proxy = weakref.proxy(c_obj)
proxy # <weakproxy at 0x... to C at 0x...>
proxy.value # 1
```

## weakref 的约束

并非所有对象都可以弱引用。支持弱引用的对象包括**类实例、用 Python(但不是用C) 编写的函数、实例方法、set、frozensets、一些文件对象、生成器、类型对象、套接字、数组、deque、正则表达式模式对象和代码对象**。

一些内置类型(如 `str`，`list` 和 `dict` )不直接支持弱引用，但可以通过子类化添加支持:

```python
class Dict(dict):
    pass

obj = Dict(red=1, green=2, blue=3)   # 该对象是可以被弱引用的
```

其他内置类型(如 `tuple` 和 `int` )即使在子类化时也不支持弱引用。

## weakref 的应用场景

### 实现缓存机制

在实际应用中，我们经常需要缓存一些对象以提高性能，但又不希望这些对象永久占用内存。这时候可以利用 `weakref` 来实现一个弱引用的缓存机制，确保缓存中的对象在内存不足时可以被自动回收。

```python
import weakref

class Cache:
    def __init__(self):
        self._cache = weakref.WeakValueDictionary()

    def get(self, key):
        return self._cache.get(key)

    def set(self, key, value):
        self._cache[key] = value

cache = Cache()
obj = A(10)
cache.set('item', obj)

print(cache.get('item'))  # <__main__.A object at 0x...>
del obj
print(cache.get('item'))  # None
```

这种机制在需要处理大量临时对象、避免内存泄漏的场景中非常有用。

### 使用弱引用集合

`weakref` 模块还提供了 `WeakSet` 类，用于实现一个弱引用的集合。它与普通集合类似，但其中的元素会以弱引用形式存在。

```python
import weakref

class A:
    def __init__(self, value):
        self.value = value

obj1 = A(10)
obj2 = A(20)

my_set = weakref.WeakSet([obj1, obj2])
print(obj1 in my_set)  # True
del obj1
print(my_set) # {<weakref at 0x...; to 'A' at 0x...>}
```

这种弱引用集合在需要跟踪大量对象但又不希望阻止其被回收时特别有用。

## 总结

`weakref` 模块提供了一种有效的内存管理手段，特别适合用于需要临时缓存对象、不希望持久占用内存的场景。通过对 `weakref` 的深入理解与合理应用，我们可以在复杂系统中更好地管理对象的生命周期，避免内存泄漏，提高程序的运行效率。
