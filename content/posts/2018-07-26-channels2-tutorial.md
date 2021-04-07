---
title: 'Django Channels2.0 websocket最佳实践'
date: 2018-07-26 02:56:48
categories: 'Python'
tags: ['Django', 'Channels', 'websocket']
---

做 web 后端开发，少不了要和 websocket 打交道。之前写过一篇关于 websocket 的实践文章 --- [《[python]记录关于 websocket 的原理和使用》](https://vimiix.com/post/2018/04/02/python-websocket/) ，不过，从 GITHUB 上可以看到，**django-websocket** 这个开源项目俨然已经是一个被放弃了的坑，并且在使用的过程中确实也有很多坑，果断弃之。

今天想聊的就是目前业界大牛都在推荐的一个框架--[Channels](https://github.com/django/channels)， Channels 是针对 Django 项目的一个增强框架，它可以是的同步的 Django 项目转变为异步的项目。它可以使得 Django 项目不仅支持 HTTP 请求，还可以支持 Websocket, chat 协议，IOT 协议 ，甚至是你自定义的协议，同时也整合了 Django 的 auth 以及 session 系統等等。

<!--more-->

如果我理解错误，请参考官方说法：

> Channels augments Django to bring WebSocket, long-poll HTTP, task offloading and other async support to your code, using familiar Django design patterns and a flexible underlying framework that lets you not only customize behaviours but also write support for your own protocols and needs.

关于框架的介绍，网上目前基本都是 1.x 版本的教程。但是 Channels 在今年二月份以后就推出了 2.0 版本，我认识这个框架也是在今年三四月份了，所有一上手就搞得 2.0，1.x 版本也就没有怎么关注。

使用 2.0 的困难就是，没有一份完整的中文文章可以参考，只能一点一点摸索官方的文档。（_渣渣英文水平，只能自救，慢慢来吧_），官方的文档写的还是蛮实用的，有教程，有示例，有模块名词讲解。基本可以对照着 Django 里面的角色来对照着学习，比如 `comsumer` 就理解成 Django 里面的 `view` 等等。

好了，和我一起去认识一下它吧~~

## ASGI

认识 Channels 之前，需要先了解一下 asgi ，全名：`Asynchronous Server Gateway Interface`。它是区别于 wsgi 的一种异步服务网关接口，不仅仅只是通过 `asyncio` 以异步方式运行，还支持多种协议。完整的文档戳[这里](https://github.com/django/asgiref/blob/master/specs/asgi.rst)

关联的几个项目：

- [https://github.com/django/asgiref](https://github.com/django/asgiref) ASGI 内存中的通道层，函数的同步异步之间相互转化需要
- [https://github.com/django/daphne](https://github.com/django/daphne) 支持 HTTP，HTTP2 和 WebSocket 协议服务器，启动 Channels 的项目需要
- [https://github.com/django/channels_redis](https://github.com/django/channels_redis) Channels 专属的通道层，使用 Redis 作为其后备存储，并支持单服务器和分片配置以及群组支持。（这个项目是 Channels 的一个附属项目，配置的时候作为可选项使用，该软件包的早期版本被称为 `asgi_redis`，如果你在使用 Channels 1.x 项目，它仍可在 PyPI 下通过这个名称使用。但 `channels_redis` 仅适用于 Channels 2 项目。）

_备注：之前体验使用过一个[基于 asgi 的 web 框架 uvicon](https://vimiix.com/post/2018/02/26/first-practise-of-uvicorn/)，也是同类项目。_

## 流程图

写代码之前先来看一下网络请求流程图：(来自网络)

![channels](https://static.vimiix.com/uPic/2021-04-06/v0AlNN.jpg)

## 实践

还是和官方同步，以一个聊天室的例子说起，Channels 提供从 PYPI 直接 pip 下载安装，：

```shell
pip install -U channels==2.0.2 Django==2.0.4 channels_redis==2.1.1
```

这里，我们不仅安装了 Channels 2.0，还安装了 Django 2.0 作为我们 web 项目框架，安装了 channels_redis 作为 Channels 的后端存储通道层。

下载好包以后，把 channels 作为 Django 的一个应用（Django 新建项目就不赘述了，假设我们已经建立好另一个项目叫 **`my_project`**），添加到配置文件的 `INSTALLED_APPS`中：

- my_project/settings.py

```pytho
INSTALLED_APPS = (
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.sites',
    ...
    'channels',
    'chat'
)
```

`chat`是我们准备建立的聊天室应用，接着就建立我们的 asgi 应用，并指定其要使用的路由。在 settings 同级目录下新建一个 `routing.py` 的文件：

- my_project/routing.py

```python
from channels.auth import AuthMiddlewareStack
from channels.routing import ProtocolTypeRouter, URLRouter

import chat.routing

application = ProtocolTypeRouter({
    # (http->django views is added by default)
    # 普通的HTTP请求不需要我们手动在这里添加，框架会自动加载过来
    'websocket': AuthMiddlewareStack(
        URLRouter(
            chat.routing.websocket_urlpatterns
        )
    ),
})
```

`chat.routing`  以及  `chat.routing.websocket_urlpatterns`  是我们后面会自己建立的模块。

紧接着，我们需要在 Django 的配置文件中继续配置 Channels 的 asgi 应用和通道层的信息：

- my_project/settings.py

```
ASGI_APPLICATION = "my_project.routing.application" # 上面新建的 asgi 应用
CHANNEL_LAYERS = {
    'default': {
    	# 这里用到了 channels_redis
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            'hosts': [('127.0.0.1', 6379)], # 配置你自己的 redis 服务信息
        },
    }
}
```

到这里，Channels 的基本配置就差不多了。

我们说以官方给的聊天室的例子来说明，所以最开始的时候，在 `INSTALLED_APPS` 中，我们添加了一个 `chat`的应用。下面就来创建这个应用。

channels 的应用和 Django 的还是不太一样，所以就不使用 `python manage.py startapp`命令创建应用了。我们手动创建几个文件。

- chat/routing.py

```python
from django.urls import path

from . import consumers

websocket_urlpatterns = [ # 路由，指定 websocket 链接对应的 consumer
    path('ws/chat/<str:room_name>/', consumers.ChatConsumer),
]
```

在 routing 文件中，我们定义了一个 `websocket_urlpatterns` 的路由列表，并设定了 `ChatCunsumer` 这个消费类（理解成 Django 的`view`）。这个 routing 文件就是上面 `my_project/routing.py` 中引用到的路由。

下面来看看 consumer 的实现：

```python
import json

from asgiref.sync import async_to_sync
from channels.generic.websocket import WebsocketConsumer


class ChatConsumer(WebsocketConsumer):
    def connect(self):
        # 当 websocket 一链接上以后触发该甘薯
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = 'chat_%s' % self.room_name

        # 把当前链接添加到聊天室
        # 注意 `group_add` 只支持异步调用，所以这里需要使用`async_to_sync`转换为同步调用
        async_to_sync(self.channel_layer.group_add)(
            self.room_group_name,
            self.channel_name
        )

		# 接受该链接
        self.accept()

    def disconnect(self, close_code):
        # 断开链接是触发该函数
        # 将该链接移出聊天室
        async_to_sync(self.channel_layer.group_discard)(
            self.room_group_name,
            self.channel_name
        )

    def receive(self, text_data):
        # 前端发送来消息时，通过这个接口传递
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        # 发送消息到当前聊天室
        async_to_sync(self.channel_layer.group_send)(
            self.room_group_name,
            {
                # 这里的type要在当前类中实现一个相应的函数，
                # 下划线或者'.'的都会被Channels转换为下划线处理，
                # 所以这里写 'chat.message'也没问题
                'type': 'chat_message',
                'message': message
            }
        )

    # 从聊天室拿到消息，后直接将消息返回回去
    def chat_message(self, event):
        message = event['message']

        # Send message to WebSocket
        self.send(text_data=json.dumps({
            'message': message
        }))
```

以上就是以同步的简单的实现了聊天室的收发功能，我们来看看这个互动过程是怎样进行的。

## 请求链接

当前端发起一个 `websocket` 的请求过来的时候，会自动触发 `connect()`函数事件。

前端发起请求的实现方式：

- chat/templates/chat/room.html

```html
<script>
  var roomName = {{ room_name_json }};

  var chatSocket = new WebSocket(
      'ws://' + window.location.host + '/ws/chat/' + roomName + '/');
  ...
</script>
```

触发 `connect`事件以后，进入函数，`self.scope`可以类比的理解为 django 中的`self.request`，从 url 中取出`room_name`字段备用，这里的变量名是在路由中设置的。

chat/urls.py

```python
from django.urls import path

from . import views

urlpatterns = [
    path('', views.index, name='index'),
    path('<str:room_name>/', views.room, name='room'),
]
```

`self.channel_name` 是每一个 websocket 请求过来时候， Channels 自动帮我们生成的一个个性化名称，实现代码就是用随机函数组装：

- channels/layers.py : L243

```python
async def new_channel(self, prefix="specific."):
        """
        Returns a new channel name that can be used by something in our
        process as a specific channel.
        """
        return "%s.inmemory!%s" % (
            prefix,
            "".join(random.choice(string.ascii_letters) for i in range(12)),
        )
```

然后我们将新的 `channel_name` 和 `room_name` 绑定，也相当于，将当前链接加入到指定聊天室。

## 端口链接

前端实现方式：

- chat/templates/chat/room.html

```html
<script>
  ....
  chatSocket.onclose = function(e) {
      console.error('Chat socket closed unexpectedly');
  };
  ....
</script>
```

当 server 端的 WebSocket 调用 `self.close()` 关闭时，前端的  `chatSocket.onclose`  会自动触发。如果前端页面关闭，导致端口链接时，后端的`disconnect()`函数会自动触发。

## 接收信息

前端实现方式：

- chat/templates/chat/room.html

```html
<script>
  ....
  // 点击按键，发送信息是很常见的逻辑
  document.querySelector('#chat-message-submit').onclick = function(e) {
      var messageInputDom = document.querySelector('#chat-message-input');
      var message = messageInputDom.value;
      chatSocket.send(JSON.stringify({
          'message': message
      }));

      messageInputDom.value = '';
  };
</script>
```

`chatSocket.send`  会将信息发送到服务端，服务端的`receive()`  事件将会自动触发，将收到的 `message` 送到对应的 group 中，

`type` 字段是一个可以被调用的对象名，Channels 会根据这个名称调度相应的方法来处理逻辑。

当在相应的函数中处理结束以后，调用`send()`方法将消息传递回前端。

前端接收信息的方式：

```html
<script>
  ....
  chatSocket.onmessage = function(e) {
      var data = JSON.parse(e.data);
      var message = data['message'];
      document.querySelector('#chat-log').value += (message + '\n');
  };
  ....
</script>
```

`chatSocket.onmessage`  会接收到后端发来的信息，前端再将该信息渲染到页面上。

## 异步写法

上面是同步方式的写法，官方还给出了异步的写法，在我们的项目中，也采用了异步的写法，Channels 对于异步的支持是非常好的。

代码如下：

```python
import json
from channels.generic.websocket import AsyncWebsocketConsumer


class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope['url_route']['kwargs']['room_name']
        self.room_group_name = 'chat_%s' % self.room_name

        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )

        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )

    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']

        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'chat_message',
                'message': message
            }
        )

    async def chat_message(self, event):
        message = event['message']

        await self.send(text_data=json.dumps({
            'message': message
        }))
```

## 从任意地方推送消息

仅仅只了解单次的收发，可能在项目中不能满足真正的需求。假如我们在项目中，其他的地方也需要向前端推送消息，怎么办？

这里就体现出了使用 Channel Layers 的好处。Channel Layers 因为是使用 redis 作为消息通道层，这使得一个应用中两个实例之间可以实现通信。如果你不想通过数据库来传递所有消息或事件，那么 Channel Layers 是开发分布式实时应用程序的非常好的实践。

有了这个，我们就可以项目的任何地方（脱离了 `consumer` 以外），给通道内广播信息。我们从上面可以知道，要想调用 `group_send` 需要有 `self.channel_layer`的实例。但在脱离了 consumer 的情况下， Channels 提供了 `get_channel_layer` 函数接口来获取它。

```python
from channels.layers import get_channel_layer
channel_layer = get_channel_layer()
```

然后，一旦你获取到了它，你就可以调用它上面的方法。但请记住，**channel layers**仅支持异步方法，因此你可以从自己的异步上下文中调用它：

```python
for chat_name in chats:
    await channel_layer.group_send(
        chat_name,
        {"type": "chat.system_message", "text": announcement_text},
    )
```

或者，如果你的上下文是同步环境，可以使用 `async_to_sync` 来转换调用：

```python
from asgiref.sync import async_to_sync

async_to_sync(channel_layer.group_send)("chat", {"type": "chat.force_disconnect"}
```

—— EOF ——
