---
title: 'vmx-manager:基于django框架开发的cmdb系统'
date: 2017-07-18 22:58:21
tags:
  - Django
  - Python
  - cmdb
categories: Project
---

## 系统介绍

VMX-Manager 是一个基于 Django 框架开发的一款 CMDB 管理系统，目前处于 beta 版本，功能主要包含: 管理系统管理的客户主机列表， 动态显示客户机状态信息，远程指令操作指定客户机以及主机与客户机之间的文件传输。

<!--more-->

###### 注：VMX 取自作者昵称`Vimiix`缩写

## 功能描述：

1. 云主机的信息统计
2. 数据的动态展示
3. 远程指令控制
4. 文件传输

## 开发环境

- win10
- python2.7
- Django1.11.1
- paramiko2.2.1
- Sqlite3

## 整体架构

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/flow.png)

## 使用说明及展示

###### _注：本次使用说明基于在本地 IP 端口，直接访问`http://127.0.0.1:8000`即可_

### 开启系统

第一种方法：

- 命令行切换到项目的根目录（`/vmx_manager`）下,执行命令`python manager.py runserver 0.0.0.0:8000` ，这里让系统启动并设置 IP 为`0.0.0.0`目的是可以让客户端主机访问。

第二种方法：

- 直接将本项目导入 pycharm,运行启动项目即可。

### 管理员注册

新人访问系统，需首先注册一个管理员账号，注册地址:`http://127.0.0.1:8000/manager/register/`

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/register.png)

### 登录

注册成功以后，会自动跳转至登录页面，用刚才注册的账号登录即可。

若不想注册，直接使用此账号登陆：

- 账号：`admin`
- 密码：`123456`

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/vmx-manager-login.png)

进入首页，可以看到本系统的基本介绍和功能选择界面

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/index.png)

自登录进来以后，会自动保存管理员信息在本地 cookie 中，有效期是 1 小时，也就是说 1 小时以后，系统会要求管理员重新登录，否则自动退出。

下面就依次介绍每个功能

### 主机列表

在这个页面，展示了目前已经在管理的客户主机信息,其中包含`主机的IP`，`自定义名称`，`操作系统`，`CPU型号`，以及`最近的一次重启时间`。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/host.png)

在这个页面，管理员可以直接操作删除目前管理的客户主机，将不再对其监控。也可以添加一个新的主机进来，点击`新增`按钮，跳转至添加页面，将要添加的主机 IP，可以管理员信息填写进去，系统会自动去读取要管理的主机相关信息，并保存到数据库，添加成功弹窗提示，点击`确定`返回到主机列表，此时会看到新添加成功的主机。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/add_host.png)

### 动态信息

这个页面作了一个信息动态的展示，因为要想显示更多的信息，只是在脚本中，去多读取一些信息，发送回来，分图表去展示，这里目前只收集到一个客户机发送上来的 CPU 信息用来做例子展示。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/cpu.png)

> <font color='red'>这个页面想要看到数据的动态变化，需要讲本项目根目录下的`sendData.py`文件打开，配置好服务器的 IP(比如我 win10 的 IP 是`192.168.1.109`)，并使用一会儿要说到的第四个文件分发功能将这个文件，下发到客户机上去运行起来。这样就会在这个页面看到对应客户主机的动态信息。</font>

### 远程指令

在这个页面有两个功能：

1. 给指定远程主机发生指令，并获取执行结果
2. 查看最近执行成功的 8 条指令

##### 发送指令

下面的例子中，我向我的 AWS 云主机发送了一条指令`ls /home/ec2-user`，查看我的用户目录下的文件信息,右边的黑色框中回显了收到的信息。若链接主机失败，会在右侧的显示框，提示链接不成功。
![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/remote.png)

#### 发送指令历史

点击`发送历史`按钮，会在右侧框中显示历史纪录，格式为：`发送时间>>发送指令>>远程主机IP`

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/history.png)

### 文件传输

这个页面同样也是有两个功能：：

1. 向服务器上传文件
2. 向指定 IP 的远程主机，分发文件

成功则在本页面提示发送成功，失败提示发送失败

具体使用请参考下面的图片：

##### 上传文件

上传成功以后，会保存在项目根目录下的`upload`中。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/upload.png)

##### 分发文件

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/put.png)

我们到`192.168.59.129`这个 IP 的主机中去看一下，文件是否发送成功？

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/verify.png)

从上图可以看出，主机 IP`192.168.59.129`成功接收到了我们刚刚从 win10 发送的`requirements.txt`的文件。

### 写在最后

这个系统不能算成熟的产品，因为其中很多功能其实可以用更好的方式去优化实现。但是，这是一次很有收获的经历。在写专注的写代码的三天中，从前端到后端到数据库，遇到了不计其数的坑，但都通过自己不断的 google,思考，优化，一步一步的都解决了。希望自己一如既往的保持一颗求知的心，去不断的探索新的知识，大胆的尝试，勇敢的试错，跨过去一道坎的心情是难以在黎明前去阐述的，小小感慨，聊以自勉。

### End

See project at [https://github.com/vimiix/vmx_manager](https://github.com/vimiix/vmx_manager)
