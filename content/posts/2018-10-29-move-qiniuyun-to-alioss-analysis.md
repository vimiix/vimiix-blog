---
title: '从七牛云到阿里云的自动化迁移代码说明'
date: 2018-10-29 23:22:48
categories: 'Project'
tags: ['Python', 'tool']
---

前几天叙事性的写了[一篇](https://vimiix.com/post/2018/10/24/move-qiniuyun-to-alioss/)，有点像日记，这篇分析一下代码逻辑，便于以后回顾。

### 工具

- python3.5
- 顺手的 IDE（轻量级推荐[vscode](https://code.visualstudio.com/)）
- [qiniu/qshell](https://github.com/qiniu/qshell#%E4%B8%8B%E8%BD%BD) （本文是基于 mac 系统开发，所以使用 qshell 的 mac 版本，读者请根据自己的系统下载，也可以直接跳至文末，下载本文源码，源码中 tool 文件夹中已经下载好了 mac 下的工具。这个工具只有一步使用到，所以如果只是使用一下，可以不用研究他的所有指令）
<!--more-->

### 环境搭建

1）创建虚拟环境

为了更好更干净的维护项目，我们首先需要创建一个独立的 Python 虚拟环境，这里我们要创建的是 Python3.5 的虚拟环境，平时我些项目会使用 3.6 但由于本次项目中所要使用一个 oss2 包官方最高支持 3.5，就只能使用 3.5 了。（这里需要确保你电脑系统中已经安装过对应版本的 Python 解释器。如果没有后面创建虚拟环境会找到继承的版本，请自行下载安装。）

先使用系统的自带的 pip 安装 [virtualenv](https://virtualenv.pypa.io/en/latest/)，这是一个创建虚拟环境的 Python 第三方工具包.

```bash
pip3 install virtualenv
```

安装成功以后，应该会自动将 virtualenv 指令添加到环境变量中，在任意路径使用。

这时候我们新建一个空的目录（名字根据自己的喜好取），作为工作目录并切换到目录下：

```bash
mkdir move_qiniuyun_to_alioss
cd move_qiniuyun_to_alioss
```

进入工作目录以后，创建一个 python3.5 的虚拟环境：

```bash
virtualenv  --python=python3.5 venv --no-site-package
```

执行完上面的指令后，会在当前目录生成一个 venv 的文件夹，这个就是我们要使用的虚拟环境。指令解释: --python 是用来指定虚拟环境所要继承的 Python 解释器版本。--no-site-package 是指定在创建虚拟环境的时候，只安装必要的三个包，不继承其他的包，保证创建的虚拟环境是一个依赖干净的环境。

2）激活虚拟环境

在当前目录下执行下面的指令，来激活虚拟环境：

```bash
source venv/bin/activate
```

当终端的提示符前面出现 (venv) ，就表示虚拟环境激活成功了。

### 编写代码

环境创建好了，接下来就可以开始正式编写代码了。

写代码之前，先看下流程框架图，便于更清晰的了解代码。

![img](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/move_qiniu_to_alioss_work_flow.jpg)

从左往右看，首先我们需要将目前七牛云已有的数据，做一份文件索引，这里需要用到前面提到的 qshell 工具，可以直接帮我们生成 listbucket.txt 文件

具体的指令如下：

```bash
# 设置本地用户访问秘钥
./tool/qshell-darwin-x64 account <ak> <sk>
# 生成索引文件
./tool/qshell-darwin-x64 listbucket2 <bucket_name> listbucket.txt
```

指令第一步中需要先设置本地的访问秘钥，这个 ak 和 sk 获取地址，请访问 <https://portal.qiniu.com/user/key> 查询你自己的秘钥即可。 第二部中的 bucket_name 是你要迁移的数据在七牛云上的 bucket 名字 。

执行上面的两步指令以后，会在当前目录中生成一个 listbucket.txt 的文件。里面的数据格式类似这样，以 Tab 分割：

Key\tSize\tHash\tPutTime\tMimeType\tFileType\tStatus\tEndUser

```
图片1.png	2190	FiQ7uGNJyTxKH_JbtHzr_ugiFbuq	14889873107405225	image/png	0	0
图片2.png	3313	FmiEMavKk5MoHBEHo89zlRA3by_4	14889873107054820	image/png	0	0
...
```

有了这个索引文件以后，接下来就是按照流程图来实现代码逻辑就好了。

创建一个 QiniuCloud 的代理类：

```python
import os
import requests


class QiniuCloud:

    def __init__(self, base_url, bucket):
        self.bucket = bucket
        self.base_url = base_url

    def request_and_save(self, filename, data_path):
        print('[-] Downloading %s' % filename)
        url = self.base_url
        url = url.endswith('/') and url or url + '/'
        resp = requests.get(url + filename)
        if resp.status_code == 404:
            print('[×] Not found file: %s' % filename)
        elif resp.status_code == 200:
            if not os.path.exists(data_path):
                os.makedirs(data_path)
            with open(data_path + '/{}'.format(filename), "wb") as f:
                f.write(resp.content)
            print('[√] File: {} saved successful.'.format(filename))
```

初始化实例的时候，我们需要将七牛云在生成 Bucket 的时候给的外链访问测试连接，以及 Bucket 名字传递进去。调用 request_and_save 方法可以将指定的文件下载下来保存在第二个参数指定的目录中。

接下来继续创建一个 AliyunOss 的代理类，让其实现上传文件的功能：

```python
import oss2


class AliOss:

    def __init__(self, ak_id, ak_secret):
        self._ak_id = ak_id
        self._ak_secret = ak_secret
        self.auth = self._auth()

    def _auth(self):
        auth = oss2.Auth(self._ak_id, self._ak_secret)
        return auth

    def bucket(self, alioss_host, bucket_name):
        return oss2.Bucket(self.auth, alioss_host, bucket_name)

    @staticmethod
    def upload(data_path, filename, bucket_instance):
        bucket_instance.put_object_from_file(
            filename,
            "{}/{}".format(data_path, filename)
        )
        print('[√] Upload file %s to alioss successful.' % filename)
```

这段代码实现了一个阿里云的代理类，其可以创建实例时需要传入阿里云 OSS 生成的 ak_id 和 ak_secret 。和七牛云一样需要秘钥，用户获取上传文件时的访问权限 。查询自己阿里云 oss 的秘钥地址，请访问：<https://usercenter.console.aliyun.com/#/manage/ak>

类中 bucket 方法，接收两个参数：alioss_host 是在阿里云创建了 bucket 以后系统会生成一个用于外网访问的 EndPoint（地域节点）链接（注意不是 bucket 链接），bucket_name 是要存放文件的 bucket 名字， 这个函数会返回一个阿里云的 bucket 实例。

上传文件的方法是 upload 静态方法，其接收的参数，我们会在后面的 `main.py` 调用时处理。注意其第三个参数 `bucket_instance` 就是需要传入该类中的 bucket 实例。

上面提到了很多需要传入的配置信息，我们统一定义一个 config.py 来存放这些信息，便于更改后不影响代码，就可以实现多个 bucket 迁移。

```python
class MyConfig:
    # 提前使用 qshell listbucket 指令生成的文件路径
    # 如果在当前文件，就直接写文件名
    listbucket_data_path = 'listbucket.txt'

    # 七牛云的外链域名
    # (例如 http://omfis13un.bkt.clouddn.com)
    qiniu_base_url = 'http://omfis13un.bkt.clouddn.com'
    # 七牛云bucket名
    qiniu_bucket = 'vimiix-blog-data'

    # ALIOSS access_key 的 id 和 secret
    # 请保密，不要上传到公开网络
    alioss_access_key_id = ''
    alioss_access_key_secret = ''

    # ALIOSS 概览中外网访问的 EndPoint（地域节点）
    alioss_host = 'oss-cn-qingdao.aliyuncs.com'
    alioss_bucket_name = 'vimiix-blog'


# 这里我们直接实例化好一个实例，供外部使用就好了，这也是单例模式的一种
myconfig = MyConfig()
```

两个代理类和配置文件写好以后，剩下的就是逻辑调度了，这一部分代码，我们放在 main.py 中集中处理。

```python
from qiniu import QiniuCloud
from alioss import AliOss
from config import myconfig


class Worker():

    def __init__(self, config):
        self.listbucket_data_path = config.listbucket_data_path
        self.data_path = "data/{}".format(config.qiniu_bucket)
        self.qiniu = QiniuCloud(
            config.qiniu_base_url,
            config.qiniu_bucket
        )
        self.alioss = AliOss(
            config.alioss_access_key_id,
            config.alioss_access_key_secret
        )
        self.alioss_bucket = self.alioss.bucket(
            config.alioss_host,
            config.alioss_bucket_name
        )

    def work(self):
        with open(self.listbucket_data_path) as f:
            for line in f:
                print('[-] Get line %s' % line.strip())
                filename = line.split('\t')[0]
                self.qiniu.request_and_save(
                    filename,
                    self.data_path
                )
                self.alioss.upload(
                    self.data_path,
                    filename,
                    self.alioss_bucket
                )


if __name__ == '__main__':
    worker = Worker(myconfig)
    worker.work()
```

代码不难，实现了一个工人类，来帮我们按照上面流程的逻辑循环干活就 OK 了。

使用这个类，就相对来说很简单， 只需要将我们在配置文件中的单实例拿来穿进去就可以了，类中运行的数据，完全取决于配置文件中的内容。

所以说，这是一个黑盒式的工具，写好以后，给使用者来说，不必关心内部是怎么实现的，只需要安装对于的字段将配置文件配置好就可以直接运行了。

### 执行代码

现在代码，都写完了，剩下的就是执行了。

在执行之前，由于我们在编写代码的过程中，引入了两个外部包 requests, oss2，所以需要在当前的虚拟环境中安装他们：

```
pip install -U requests oss2
```

静静等待执行完成就可以了。为了让别人更好的使用这个工具，我们应该将当前环境中所依赖的包声明一下，pip 工具可以帮我们快速的生成这个文件：

```
pip freeze > pip-req.txt
```

这样的话，会在当前项目中生成一个 pip-req.txt 的文本文件，里面存储了我们依赖的包以及版本号。

别人在使用的时候只需要执行下面一条指令就可以完美的搭建好依赖环境：

```
pip install -r pip-req.txt
```

对于我们项目执行的话，只需要使用虚拟环境中的 Python 解释器启动 main.py 就可以了

```
python main.py
```

### 实例结果

![img](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/result.jpg)

### 代码地址

[vimiix/move_qiniuyun_to_aliossgithub.com](https://github.com/vimiix/move_qiniuyun_to_alioss)

--- EOF ---
