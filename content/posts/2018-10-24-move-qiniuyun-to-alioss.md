---
title: '七牛云数据自动迁移到阿里云OSS'
date: 2018-10-24 19:22:48
categories: 'Project'
tags: ['Python', 'tool', 'migrate']
---

### 背景

近期收到两封七牛云发来的邮件：

> 测试域名回收通知
>
> 您的账号 xxx 在七牛云融合 CDN 加速平台有以下测试域名**还剩 7 个自然日会被系统自动回收**

由于，我博客所有的图片文件都是存储在七牛云的，这个域名也使用了一年多了，怎么突然要回收呢？

上网一搜才知道，大概是有什么不法分子之类的，使用七牛云的免费空间传播色情暴力之类的内容，被 Godday 制裁了，现在新申请的 bucket 只能使用一个月，要想绑定域名，还得备案操作。俺这小博客，也就自己玩玩的一个国外服务器，也备不了案啊。

无奈，看网上很多人都是被回收了才知道自己的图片都访问不了。还好我习惯性的看这些推送邮件，给自己留了一周时间用来备份转移。既然免费的不好用了，微博之类的图床不好迁移，所以就买了一年阿里云的 OSS 服务。

虽然我的图片还算不是很多，但要是一张一张手动下载再上传到阿里云，也是不小的工作量，而且太浪费时间了。

<!--more-->

于是，今天就花了点时间写了一个自动化迁移工具（[move_qiniuyun_to_alioss](https://github.com/vimiix/move_qiniuyun_to_alioss)），并开源到 GitHub 了，没什么复杂的操作，就是把图片 down 下来，本地备份一份数据，然后再通过阿里云的 API 接口直接上传到指定的 bucket。

因为数据量不大，也没考虑使用 FIFO，异步之类的（看情况以后再优化吧，一切以需求为导向）。

虽然小，但还是希望尽量做到通用化，我把所有的配置参数都抽到了 `config.py` 文件中。每个人根据自己的配置修改，直接就可以用了。

### 准备工作

使用 `tool/` 目录中的七牛云工具 `qshell-darwin-x64` ：

_注：qshell 使用指南请参考：[https://github.com/qiniu/qshell](https://github.com/qiniu/qshell)_

- 配置访问骑牛云的 account 的 `access_key` 和 `secret_key`
- 拉一份要搬移的七牛云 bucket 的文件清单：

```
# 需要先设置一下七牛云的 ak, sk
# 获取地址 https://portal.qiniu.com/user/key
./tool/qshell-darwin-x64 account ak sk
# 分别是 执行程序 命令行 bucket_name 生成的文件名
./tool/qshell-darwin-x64 listbucket2 vimiix-blog-data listbucket.txt
```

执行完以后正常会在当前目录生成一个 `listbucket.txt` 的文件，准备工作就做好了。

### 修改配置文件

根据 `config` 文件中的注释将每个参数设置为自己对应的值即可。

- AliOss AccessKeyID 和 AccessKeySecret 获取地址
  - <https://usercenter.console.aliyun.com/#/manage/ak>

### 执行

这个工具使用到了阿里云的 `oss2` 包，需要 版本大于 3， 但看官网写着最高支持到 Python3.5

所以我的虚拟环境也使用的 Python3.5，建议你也这么做，省的麻烦。

```bash
virtualenv --python=python3.5 venv --no-site-package

source venv/bin/activate

pip install -r pip-req.txt

python main.py
```

### 执行结果

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/result.jpg)

大功告成！

### 项目地址

- https://github.com/vimiix/move_qiniuyun_to_alioss

--- EOF ---
