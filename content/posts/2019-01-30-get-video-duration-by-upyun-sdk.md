---
title: '获取存储在又拍云CDN中视频的时长'
date: 2019-01-30 16:22:48
categories: 'note'
tags: ['note', 'video', 'CDN', '又拍云']
---

### 前置条件

- 可用的账户

- 安装又拍云 Python 版本的 SDK：

```shell
pip install upyun
```

（官方代码开源地址：[https://github.com/upyun/python-sdk/](https://github.com/upyun/python-sdk/) )

<!--more-->

### 继承官方类，编写需求代码

官方的 SDK 中只提供了一些基础的方法，可以在源码的依赖文件中看到，只依赖 `requests` 一个包，所以，官方库其实是实现了对网络交互的封装，没有实际的业务功能，真正又拍云 CDN 提供的服务，需要自己通过这个 SDK 发起 api 请求来获取。

既然这样，那就继承官方提供的 SDK，增加自己需要的方法：

```python
from upyun import UpYun
from config import BUCKET, USER, PASSWORD


class MyUpYun(UpYun):

    pass


worker = MyUpYun(BUCKET, USER, PASSWORD)

__all__ = ['worker']


```

OK，这样就实现了基本的继承，并对外提供一个 `worker` 实例，这个实例就是一个已经是登录状态的”打工仔“了。在其他模块调用 `worker` ，就可以使用又拍云 SDK 提供的工具了。

接下来就需要在我自己写的这个 `MyUpYun` 类中实现特定的功能方法了。

> 又拍云提供的全部云端服务可用移步这里查看：[http://docs.upyun.com/](http://docs.upyun.com/)

获取视频的元信息，属于 **[云处理](http://docs.upyun.com/cloud/)** 服务。

由于每次请求远程的 api 需要在 `headers` 中传递固定的签名认证，所以就需要先写一个生成签名的方法：

```python
from upyun import UpYun
from config import BUCKET, USER, PASSWORD
from upyun.modules.httpipe import cur_dt
from upyun.modules.sign import make_signature

class MyUpYun(UpYun):

    def gen_signature(self, method, uri):
        """
        args:
        	method: 请求的 HTTP 方法
        	uri: api 地址
        """
        dt = cur_dt()
        return make_signature(username=self.username,
                              password=self.password,
                              auth_server=self.auth_server,
                              method=method,
                              uri=uri,
                              date=dt)

    ...

```

这里实现了一个 `gen_signature` 方法，它会帮助我生成在请求 api 接口时所需要的加密后的签名字符串。可以看到 ，在函数内部，传递一系列 `self.xxx` 的参数，我的本地类中并没有定义这些 property，但不用担心，因为我们是继承的官方库，在实例化的时候，已经把用户认证信息穿进去了，对应的这个 property 都已经在父类里定义好了。后面类似直接用到的属性，也是这样。

到这里，签名也准备好了，接下来就是请求获取视频元信息的 api。

> 获取音视频元信息文档：[http://docs.upyun.com/cloud/sync_video/#\_14](http://docs.upyun.com/cloud/sync_video/#_14)

从文档可以看到，这里需要通过 `POST` 方法来请求 `http://p1.api.upyun.com/<service>/avmeta/get_meta`

接口。但是需要在 body 中传递一个 `source` 参数，这个 `source` 其实就是对应的文件在又拍云的 bucket 中存储的文件路径，如果你没有对你的 bucket 做目录分级，可以直接传文件名。

因为 upyun SDK 内部的 HTTP 请求都是使用的 requests 库，所以，其实请求结果拿到的 response 其实是 requests 中的 `Response` 对象，可以很方便的通过 `.json()` 来获取字典结构的数据。

下面是获取视频元数据的方法代码：

```python
class MyUpYun(UpYun):

    def get_meta(self, source):
        method, dt, headers = 'POST', cur_dt(), {}
        host = 'p1.api.upyun.com'
        uri = '/%s/%s' % (self.service, 'avmeta/get_meta')
        data = json.dumps({'source': source})
        headers['Authorization'] = self.gen_signature(method, uri)
        headers['Date'] = dt
        resp = self.hp.do_http_pipe(
            method, host, uri, value=data, headers=headers, stream=False
        )
        content = resp.json()
        return content
```

在业务层，通过调用 `worker.get_meta(source)` 即可获取到对应视频文件的元信息了。

在返回的结果中数据示例是这样的，对应到每个字段的定义官方文档中都有介绍：

```json
{
  "streams": [
    {
      "index": 0,
      "type": "video",
      "video_fps": 25,
      "video_height": 236,
      "video_width": 426,
      "codec_desc": "H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10",
      "codec": "h264",
      "bitrate": 99608,
      "duration": 184.8,
      "metadata": {
        "handler_name": "VideoHandler",
        "language": "und"
      }
    },
    {
      "index": 1,
      "type": "audio",
      "audio_channels": 2,
      "audio_samplerate": 44100,
      "codec_desc": "AAC (Advanced Audio Coding)",
      "codec": "aac",
      "bitrate": 48005,
      "duration": 184.855011,
      "metadata": {
        "handler_name": "SoundHandler",
        "language": "und"
      }
    }
  ],
  "format": {
    "duration": 184.902,
    "fullname": "QuickTime / MOV",
    "bitrate": 154062,
    "filesize": 3.560797e6,
    "format": "mov,mp4,m4a,3gp,3g2,mj2"
  }
}
```

### 获取视频时长

OK，经过上面的接口，就拿到了视频的全部元信息，接下来就是不怎么重要的回扣标题一下。

在上面的示例数据中，可以找到 `format.duration` 字段，这就是我所要找的视频时长数据。✨🍰✨

```python
meta = worker.get_meta(source)
duration = meta and meta['format']['dutation'] or None
```

这里只记录获取视频元信息这一个 api 接口的使用。回看官方文档，其实有很多很多的服务可以使用，本地封装方法也类似上面这样，就可以自己写一个比官方更富有的 SDK 啦~

共勉！

--- EOF ---
