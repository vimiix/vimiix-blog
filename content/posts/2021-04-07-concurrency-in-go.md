---
title: 'Golang的并行模式实践'
date: 2021-04-07 18:15:48
categories: 'golang'
tags:
  - 'concurrency'
  - 'golang'
  - 'goroutine'
---

## Goroutine

C#、Lua、Python 的用户可能会发现 Go 的 goroutine 和协程之间有很多相似之处，没错，从命名上也可以看出二者具有相似性。

但二者之间也有些区别：

- goroutine 隐含了并行的特性，一切交给 go 的 runtime 来实现调度，而协程需要应用自己来编写并行代码
- goroutine 通过信道（`Channel`）进行通信；协程通过 `yield` 和 `next`操作进行通信

一般来说，goroutine 比协程更强大。而且，我们可以很容易地将协程的逻辑移植到 goroutine 来获得更好的并行效果。

<!--more-->

## Generators

在 python 中，要实现一个生成器，需要通过 `yield` 生成数据，通过 `next` 获取数据：

```python
def integers():
  i = 1
  while True:
    yield i
    i+=1

integerGenerator = integers()
next(integerGenerator)
# 1
next(integerGenerator)
# 2
next(integerGenerator)
# 3
```

同样的，生成器也可以在 Go 中实现，只是使用了 `channel` 来代替 `yield`:

```go
func integers () chan int {
    yield := make (chan int);
    count := 0;
    go func () {
        for {
            yield <- count;
            count++;
        }
    } ();
    return yield
}

next := integers();
func generateInteger () int {
    return <-next
}

generateInteger()
// 0
generateInteger()
// 1
generateInteger()
// 2
```

## Iterators

go 的 channel 在语法层面就是天然的迭代器。

```go
for x := range channel {
    fmt.Println(x)
}
```

## Futures

先来看一个例子：

```go
func InverseProduct (a Matrix, b Matrix) {
    a_inv := Inverse(a);
    b_inv := Inverse(b);
    return Product(a_inv, b_inv);
}
```

在这个例子中，a 和 b 的求逆矩阵需要先被计算。那么为什么在计算 b 的逆矩阵时，需要等待 a 的逆计算完成呢？显然不必要，这两个求逆运算其实可以并行执行的。如下代码实现了并行计算方式：

```go
func InverseFuture (a Matrix) {
    future := make (chan Matrix);
    go func () {
      future <- Inverse(a)
    }();
    return future;
}

func InverseProduct (a Matrix, b Matrix) {
    a_inv_future := InverseFuture(a);
    b_inv_future := InverseFuture(b);
    a_inv := <-a_inv_future;
    b_inv := <-b_inv_future;
    return Product(a_inv, b_inv);
}
```

在这个改进版本中，InverseFuture 函数启动一个 goroutine 来执行求逆矩阵计算，并立即返回一个最终保存 future 值的 Channel，这样就实现了 a 和 b 的并行计算。

## Producer-Consumer

通过使用 goroutine 和 channel 可以很容易地实现生产者-消费者模型的设计：

```go
var done = make(chan bool)
var msgs = make(chan int)

func produce () {
    for i := 0; i < 10; i++ {
        msgs <- i
    }
    done <- true
}

func consume () {
    for {
      msg := <-msgs
      fmt.Println(msg)
   }
}

func main () {
   go produce()
   go consume()
   <- done
}
```
