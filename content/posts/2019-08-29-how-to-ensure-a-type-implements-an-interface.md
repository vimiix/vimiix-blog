---
title: 'Golang如何确保一个类型实现了某个interface'
date: 2019-08-29 16:22:48
categories: 'golang'
tags: ['note', 'interface', 'golang']
---

在 golang 中，接口（interface）代表一种『协议』存在，它是一个声明了多个方法的集合。

接口是被隐式实现的，也就是说，我们在开发中定义一个类型（type）的时候，不需要声明这个类型实现了哪个接口。在使用的时候往往通过断言来的 ok-idom 来进行类型判断该类型是否实现了目标接口，放置调用方法失败抛出 panic：

```go
if value, ok := AType.(BInterface) {
  fmt.Println("ok")
} else {
  fmt.Println("no")
}
```

如果断言失败，那么 ok 的值将会是 false,但是如果断言成功 ok 的值将会是 true,同时 value 将会得到所期待的正确的值。

但是，在某些情况下，我们可能希望明确地检查接口中哪些方法没有被实现。最好的方法就是借助编译器地检查功能。

假设定义了一个 `Programmer` 接口，一个 `Human` 类型：

[>>> 试一下](https://play.golang.org/p/Az2TON0ZDXD)

```go
package main

type Programmer interface {
  Code() string
}

type Human struct {}

func main() {}
```

这段代码没有做任何操作，可以正常地执行。

假设我们希望 `Human` 类型实现了 `Programmer` 接口，可以在代码中定义一个 `_` （下划线表示忽略这个变量）变量来实现检测效果

```go
package main

type Programmer interface {
  Code() string
}

type Human struct {}

// 利用编译器检查接口实现
var _ Programmer = (*Human)(nil)

func main() {}
```

当再次执行这段代码地时候，就会报出如下编译错误：

```bash
./prog.go:10:5: cannot use (*Human)(nil) (type *Human) as type Programmer in assignment:
	*Human does not implement Programmer (missing Code method)
```

这个时候，编译器很友好地将我们在 `Human` 类型中缺少地方法显示了出来。避免上线后出问题。其实，如果是在 Goland 类似有实时提示的 IDE 中开发的话，当这段代码被写完的时候，开发环境就会实时检测出来，甚至都不需要自己编译：

![img](https://vimiix-blog.oss-cn-qingdao.aliyuncs.com/1567154584416.jpg)

当我们给 `Human` 类型实现了对应的方法以后，就不会报错了：

```go
package main

type Programmer interface {
    Code() string
}

type Human struct {}

func (*Human) Code() string{
    return "BUG"
}

// here
var _ Programmer = (*Human)(nil)

func main() {}
```

[>>>试一下](https://play.golang.org/p/S1F-i4Ba15B)
