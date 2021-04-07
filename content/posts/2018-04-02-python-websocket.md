---
title: '[python]记录关于websocket的原理和使用'
date: 2018-04-02 19:20:48
categories: 'Python'
tags: ['Python', 'websocket', 'note', 'django']
---

## 什么是 websocket

WebSocket 是一种在单个 TCP 连接上进行[全双工](https://zh.wikipedia.org/wiki/%E5%85%A8%E9%9B%99%E5%B7%A5)通讯的协议。WebSocket 使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建**持久性**的连接，并进行双向数据传输。

Websocket 是一个持久化的协议，相对于 HTTP 这种非持久的协议来说的。

举个例子：

> HTTP 的生命周期通过 Request 来界定，也就是发送一次 Request，收到一次 Response ，那么在 HTTP1.0 中，这次 HTTP 请求就结束了
>
> 在 HTTP1.1 中进行了改进，使得有一个 keep-alive，也就是说，在一个 HTTP 连接中，可以发送多个 Request，接收多个 Response。但是请记住 Request = Response ， 在 HTTP 中永远是这样，也就是说一个 request 只能有一个 response。而且这个 response 也是被动的，不能主动发起。
>
> 而对于 websocket 来说，在 HTTP 的握手基础上建立起链接，服务器端可以主动的向客户端发送数据。

<!--more-->

## 为什么要使用 websocket

现在，很多网站为了实现页面上的数据更新，都是通过 ajax 技术[轮询](https://zh.wikipedia.org/wiki/%E8%BC%AA%E8%A9%A2)去服务器拉取数据。轮询是在特定的时间间隔（如每 1 秒），由浏览器对服务器发出 HTTP 请求，然后由服务器返回最新的数据给客户端的浏览器。这种传统的模式带来很明显的缺点，即浏览器需要不断的向服务器发出请求，然而 HTTP 请求可能包含较长的头部，其中真正有效的数据可能只是很小的一部分，显然这样会浪费很多的带宽等资源。

而比较新的技术去做轮询的效果是[Comet](<https://zh.wikipedia.org/wiki/Comet_(web%E6%8A%80%E6%9C%AF)>)。Comet 技术又可以分为长轮询和流技术。长轮询改进了上述的轮询技术，减小了无用的请求。它会为某些数据设定过期时间，当数据过期后才会向服务端发送请求；这种机制适合数据的改动不是特别频繁的情况。流技术通常是指客户端使用一个隐藏的窗口与服务端建立一个 HTTP 长连接，服务端会不断更新连接状态以保持 HTTP 长连接存活；这样的话，服务端就可以通过这条长连接主动将数据发送给客户端；流技术在大并发环境下，可能会考验到服务端的性能。这两种技术都是基于请求-应答模式，都不算是真正意义上的实时技术；它们的每一次请求、应答，都浪费了一定流量在相同的头部信息上，并且开发复杂度也较大。

在这种情况下，HTML5 定义了 WebSocket 协议，能更好的节省服务器资源和带宽，并且能够更实时地进行通讯。

Websocket 使用 ws 或 wss 的统一资源标志符，类似于 HTTPS，其中 wss 表示在 TLS 之上的 Websocket。如：

```
ws://example.com/wsapi
wss://secure.example.com/
```

Websocket 使用和 HTTP 相同的 TCP 端口，可以绕过大多数防火墙的限制。默认情况下，Websocket 协议使用 80 端口；运行在 TLS 之上时，默认使用 443 端口。

WebSocket 目前由 W3C 进行标准化。WebSocket 已经受到 Firefox 4、Chrome 、Opera 10.70 以及 Safari 5 等浏览器的支持。

## 一个典型的 websocket 的握手请求

- 客户端请求：

```
GET / HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Host: example.com
Origin: http://example.com
Sec-WebSocket-Key: sN9cRrP/n9NdMgdcy2VJFQ==
Sec-WebSocket-Version: 13
```

- 服务端回应：

```
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: fFBooB7FAkLlXgRSz0BT3v4hq5s=
Sec-WebSocket-Location: ws://example.com/
```

### 字段说明

- Connection 必须设置 Upgrade，表示客户端希望连接升级。
- Upgrade 字段必须设置 Websocket，表示希望升级到 Websocket 协议。
- Sec-WebSocket-Key 是随机的字符串，服务器端会用这些数据来构造出一个 SHA-1 的信息摘要。把“Sec-WebSocket-Key”加上一个特殊字符串“258EAFA5-E914-47DA-95CA-C5AB0DC85B11”，然后计算 SHA-1 摘要，之后进行 BASE-64 编码，将结果做为“Sec-WebSocket-Accept”头的值，返回给客户端。如此操作，可以尽量避免普通 HTTP 请求被误认为 Websocket 协议。
- Sec-WebSocket-Version 表示支持的 Websocket 版本。RFC6455 要求使用的版本是 13，之前草案的版本均应当弃用。
- Origin 字段是可选的，通常用来表示在浏览器中发起此 Websocket 连接所在的页面，类似于 Referer。但是，与 Referer 不同的是，Origin 只包含了协议和主机名称。
- 其他一些定义在 HTTP 协议中的字段，如 Cookie 等，也可以在 Websocket 中使用。

## 如何使用 websocket

关于如何在 Python 后端使用 websocket 的技术有多种方式，可以参考这篇文章[《python 使用 websocket 的几种方式》](https://letus.club/2016/04/10/python-websocket/)

我们的项目使用的是 Django 的框架，我这次的实践主要围绕 Django 的 websocket 插件 **dwebsocket** 展开记录。[源码地址](https://github.com/duanhongyi/dwebsocket)

### 安装 dwesocket

话不多说， pip 大法。

```
pip install dwesocket
```

### 使用

如果希望一个 view 视图既可以处理 HTTP 请求，也可以处理 websocket 请求，只需要将这个 view 函数用装饰器 `@accept_websocket` 包裹即可。也可以使用 `@require_websocket` 装饰器，这样的话这个视图只允许接受 websocket 请求，会拒绝正常的 HTTP 请求。

如果你希望在应用程序中为所有的 url 提供 websocket 请求支持，则可以使用中间件。只需要将`dwebsocket.middleware.websocketmiddleware` 添加到设置中的的 `MIDDLEWARE_CLASSES` 中。 （如果没有需要自己定义，不定义会报错，这个有点尴尬，对于这一点的优化，我已经对主库提了一个**[Pull Request](https://github.com/duanhongyi/dwebsocket/pull/35)**,希望作者还在维护....）

```python
MIDDLEWARE_CLASSES = ['dwebsocket.middleware.WebSocketMiddleware']
```

如果允许每个单独的视图接受 websocket 请求，在 settings 中设置 `WEBSOCKET_ACCEPT_ALL` 为 `True`.

```python
WEBSOCKET_ACCEPT_ALL = True
```

### 接口和属性

1. **request.is_websocket()**

   如果是个 websocket 请求返回 True，如果是个普通的 http 请求返回 False,可以用这个方法区分它们。

2. **request.websocket**

   在一个 websocket 请求建立之后，这个请求将会有一个 websocket 属性，用来给客户端提供一个简单的 api 通讯，如果 request.is_websocket()是 False，这个属性将是 None。

3. **WebSocket.wait()**

   返回一个客户端发送的信息，在客户端关闭连接之前他不会返回任何值，这种情况下，方法将返回 None

4. **WebSocket.read()**

   如果没有从客户端接收到新的消息，read 方法会返回一个新的消息，如果没有，就不返回。这是一个替代 wait 的非阻塞方法

5. **WebSocket.count_messages()**

   返回消息队列数量

6. **WebSocket.has_messages()**

   如果有新消息返回 True，否则返回 False

7. **WebSocket.send(message)**

   向客户端发送消息

8. **WebSocket.**iter**()**

   websocket 迭代器

### 实践例程

从客户端接收一条消息，将该消息发送回客户端并关闭连接（通过视图返回）：

```python
from dwebsocket import require_websocket

@require_websocket
def echo(request):
    message = request.websocket.wait()
    request.websocket.send(message)
```

我们也可以让服务端不自动关闭连接,下面的例程中，服务器端会将客户端发来的消息转为小写发送回去，并增加了普通的 HTTP 请求的响应，也做同样操作返回：

```python
from django.http import HttpResponse
from dwebsocket import accept_websocket

def modify_message(message):
    return message.lower()

@accept_websocket
def lower_case(request):
    if not request.is_websocket(): # 普通HTTP请求
        message = request.GET['message']
        message = modify_message(message)
        return HttpResponse(message)
    else: # websocket请求
        for message in request.websocket:
            message = modify_message(message)
            request.websocket.send(message)
```

### 改变 dwebsocket 后端

目前 dwebsocket 支持两种后端，分别是 **default** 的和 **uwsgi**。

默认 **default** 支持 **Django 自身的开发服务器**, **eventlent**, **gevent**, 和**gunicore**。

如果你想使用**uwsgi**后端，在**settings.py**文件中添加 **WEBSOCKET_FACTORY_CLASS**：

```python
WEBSOCKET_FACTORY_CLASS = 'dwebsocket.backends.uwsgi.factory.uWsgiWebSocketFactory'
```

运行 uwsgi ：

```bash
uwsgi --http :8080 --http-websockets --processes 1 \
--wsgi-file wsgi.py--async 30 --ugreen --http-timeout 300
```

## websocket 的优缺点

### 优点

- 服务器与客户端之间交换的标头信息很小，大概只有 2 至 10 字节（和数据包长度有关）;

- 客户端与服务器都可以主动传送数据给对方;

- 不用频率创建 TCP 请求及销毁请求，减少网络带宽资源的占用，同时也节省服务器资源;

- 更强的实时性。由于协议是全双工的，所以服务器可以随时主动给客户端下发数据。相对于 HTTP 请求需要等待客户端发起请求服务端才能响应，延迟明显更少;

- 更好的二进制支持。Websocket 定义了二进制帧，相对 HTTP，可以更轻松地处理二进制内容;

- 可以支持扩展。Websocket 定义了扩展，用户可以扩展协议、实现部分自定义的子协议。如部分浏览器支持压缩等。

### 缺点

下面这些出自知乎，优点鸡肋的缺点，就是对开发者技术要求高点应该不算缺点，新的技术总有个普及的过程。

对前端开发者：

- 往往要具备数据驱动使用 javascript 的能力，且需要维持住 ws 连接（否则消息无法推送）

对于后端开发者：

- 一是长连接需要后端处理业务的代码更稳定（不要随便把进程和框架都 crash 掉）

- 二是推送消息相对复杂一些

- 三是成熟的 http 生态下有大量的组件可以复用，websocket 比较新

## 参考资料

- [1][https://zh.wikipedia.org/wiki/websocket](https://zh.wikipedia.org/wiki/WebSocket)

- [2][https://github.com/duanhongyi/dwebsocket](https://github.com/duanhongyi/dwebsocket)

- [3][websocket 协议有哪些劣势和缺点？](https://www.zhihu.com/question/20155314)
