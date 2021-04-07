---
title: '[Python]Uvicorn初体验'
date: 2018-02-26 12:20:48
categories: 'Python'
tags: ['uvicorn', 'web', 'Python', 'asyncio']
---

## uvicorn 简介

`uvicorn`是一个基于`asyncio`开发的一个轻量级高效的 web 服务器框架。

> 官网：[http://www.uvicorn.org](http://www.uvicorn.org)

`uvicorn` 设计的初衷是想要实现两个目标：

- 使用[`uvloop`](https://github.com/MagicStack/uvloop)和[`httptools`](https://github.com/MagicStack/httptools)实现一个极速的`asyncio`服务器。
- 实现一个基于[`ASGI(异步服务器网关接口)`](http://channels.readthedocs.io/en/stable/asgi.html)的最小的应用程序接口。

它目前支持`http`，`websockets`，`Pub/Sub` 广播，并且可以扩展到其他协议和消息类型。

<!--more-->

## 安装使用

`uvicorn` 仅支持 python 3.5.3 以上版本，我们可以通过 pip3 来快速的安装。

###### Tip：建议和我一样，直接使用`pip3`来安装，就不用关心系统默认版本了。

```python
➜ pip3 install uvicorn
	...
Successfully installed gunicorn-19.7.1 httptools-0.0.10
uvicorn-0.0.15 uvloop-0.9.1 websockets-4.0.1
```

安装成功以后。就可以来编写我们的服务器应用代码了。

先创建一个应用文件`app.py`（名字可以自取）

在这个文件中，来编写一个简单的服务器应用。

```python
1 # coding:utf-8
2
3 async def hello_world(message, channels):
4     content = b'<h1>Hello World</h1>'
5     resp = {
6         'status': 200,
7         'headers': [[b'content-type', b'text/html'],],
8         'content': content,
9     }
10     await channels['reply'].send(resp)
11
```

写好以后，先来尝试运行一下，跑通了再看代码中具体内容的含义。

运行方式是： `uvicorn 文件名:callable对象名`

```python
➜ uvicorn app:hello_world
```

提示下面的内容就表示服务器启动成功了

（信息中包括了访问地址和端口号，以及 worker 运行的线程 id）

```python
[2018-02-26 00:48:52 +0800] [55984] [INFO] Starting gunicorn 19.7.1
[2018-02-26 00:48:52 +0800] [55984] [INFO] Listening at: http://127.0.0.1:8000 (55984)
[2018-02-26 00:48:52 +0800] [55984] [INFO] Using worker: uvicorn 0.0.15
[2018-02-26 00:48:52 +0800] [55987] [INFO] Booting worker with pid: 55987

```

这时候我们在浏览器中访问`http://127.0.0.1:8000`,会看到网页上显示出 `h1` 号字体的`Hello World`，也就是我们代码中定义的 `content` 的字符串内容。

OK，服务器跑起来了，接下来，我们来看一下代码是如何将内容返回给浏览器的。

## 接口分析

在代码中我们定义了一个协程函数，ASGI 的协议要求应用应该对外暴露一个可接受 `message` 和 `channels` 这两个参数的协程可调用对象(callable)：

- `message`：一个 ASGI 消息（但有区别，见下文）
- `channels`：一个字典（ `<unicode string> : <channel interface>` ）

`<channel interface>` 是具有以下属性的对象：

- .send(message) 一个协程，用于发送返回的消息。可选
- .receive() 一个协程，用于接收进来的消息。可选
- .name 一个`unicode`字符串，channel 的唯一标识。可选

uvicorn 中的`message`区别于 ASGI 中的消息：

- 消息还包括一个`channel`关键字，区别消息类型，例如:`'channel':'http.request'`
- 消息不包括例如`reply_channel`或`body_channel`这样的`channel`名称，，而是`channels`字典可以查看允许的`channel`类型。

#### 举例：

传进来的一个 HTTP 请求可能会是类似下面这样的`message`和 `channels`.

`message`

```python
{
    'channel': 'http.request',
    'scheme': 'http',
    'root_path': '',
    'server': ('127.0.0.1', 8000),
    'http_version': '1.1',
    'method': 'GET',
    'path': '/',
    'headers': [
        [b'host', b'127.0.0.1:8000'],
        [b'user-agent', b'curl/7.51.0'],
        [b'accept', b'*/*']
    ]
}
```

`channels`

```python
{
    'reply': <ReplyChannel>
}
```

为了做出响应，应用程序需要向`reply`的`channel`发送（`.send()`）一个 http 响应，例如：

```python
await channels['reply'].send({
    'status': 200,
    'headers': [
        [b'content-type', b'text/plain'],
    ],
    'content': b'Hello, world'
})
```
