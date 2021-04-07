---
title: 'DevOps|SaltStack的部署和基本指令'
date: 2017-06-27 20:08:03
tags:
  - Python
  - SaltStack
  - DevOps
categories: DevOps
---

SaltStack 是一个服务器基础架构集中化管理平台，具备配置管理、远程执行、监控等功能，。SaltStack 基于 Python 语言实现，结合轻量级消息队列（ZeroMQ）与 python 第三方模块（Pyzmq、PyCrypto、Pyjinjia2、python-msgpack 和 PyYAML 等）构建。

<!--more-->

通过部署 SaltStack 环境，可以在成千上万台服务器上做到批量执行命令，根据不同业务特性进行配置集中化管理、分发文件、采集服务器数据、操作系统基础及软件包管理等，SaltStack 是运维人员提高工作效率、规范业务配置与操作的利器。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/SaltStack.png)

# 部署与基本配置

**我的系统环境**：

- 系统：CentOS-7-x64
- yum 源:网易 163yum 源
- python：2.7.13
- masterIP：192.168.1.107
- minionIP：192.168.1.107(我直接在一个虚拟机内安装 minion)

## step 0：安装 epel yum 源

```
	yum install -y epel-release
	yum clean all
	yum makecache
```

## step 1：安装 master 和 minion

**master**

```
	yum install -y salt-master
```

**minion**

```
	yum install -y salt-minion
```

> 安装的同时:
> 1、关闭防火墙
> `service iptables stop`
> 2、关闭 SELinux
> `setenforce 0`

## step 2：基本配置

> <font color="red">注意：下面的配置文件均严格遵循 yaml 文件格式，冒号后面必须有空格</font>

**master**

`vim`打开`/etc/salt`目录下面的`mater`文件

1、在命令模式下搜索关键词`interface`，找到下面段落，复制一行在下面打开注释并指定 master 的主机 IP 地址

```
	# The address of the interface to bind to:
	#interface: 0.0.0.0
	interface: 192.168.1.107
```

2、同样，搜索关键词`auto_accept`，指定是否自动接收 minion 端，这里设置为`true`

```
	# Enable auto_accept, this setting will automatically accept all incoming
	# public keys from the minions. Note that this is insecure.
	#auto_accept: False
	auto_accept: True

```

3、最后搜索关键词`file_root`,设置配置文件的根目录，只需打开下面段落的最后三行注释即可

```
	# The file server works on environments passed to the master, each environment
	# can have multiple root directories, the subdirectories in the multiple file
	# roots cannot match, otherwise the downloaded files will not be able to be
	# reliably ensured. A base environment is required to house the top file.
	# Example:
	# file_roots:
	#   base:
	#     - /srv/salt/
	#   dev:
	#     - /srv/salt/dev/services
	#     - /srv/salt/dev/states
	#   prod:
	#     - /srv/salt/prod/services
	#     - /srv/salt/prod/states
	#
	file_roots:
	  base:
	    - /srv/salt
```

**minion**

`vim`打开`/etc/salt/`目录下面的`minion`文件

1、搜索`master:`关键词,配置所有连接的服务器 IP(_没有修改过的话在第 16 行_)

```
	# Set the location of the salt master server. If the master server cannot be
	# resolved, then the minion will fail to start.
	#master: salt
	master: 192.168.1.107
```

2、搜索`id:`关键词，设置 minion 在 master 端的名称

```
	# Explicitly declare the id for this minion to use, if left commented the id
	# will be the hostname as returned by the python call: socket.getfqdn()
	# Since salt uses detached ids it is possible to run multiple minions on the
	# same machine but with different ids, this can be useful for salt compute
	# clusters.
	#id:
	id: local-minion
```

## step 3：重启 master 和 minion

**重启 master**

```
	systemctl restart salt-master.service
```

重启以后可以查看一下 master 服务当前状态,显示`active`即启动成功

```
	[root@localhost vimiix]# systemctl status salt-master.service
	● salt-master.service - The Salt Master Server
	   Loaded: loaded (/usr/lib/systemd/system/salt-master.service; disabled; vendor preset: disabled)
	   Active: **active** (running) since Tue 2017-06-27 19:19:28 CST; 1min 22s ago
	 Main PID: 49362 (salt-master)
	   CGroup: /system.slice/salt-master.service
	           ├─49362 /usr/bin/python /usr/bin/salt-master
	           ├─49377 /usr/bin/python /usr/bin/salt-master
	           ├─49378 /usr/bin/python /usr/bin/salt-master
	           ├─49379 /usr/bin/python /usr/bin/salt-master
	           ├─49380 /usr/bin/python /usr/bin/salt-master
	           ├─49382 /usr/bin/python /usr/bin/salt-master
	           ├─49383 /usr/bin/python /usr/bin/salt-master
	           ├─49384 /usr/bin/python /usr/bin/salt-master
	           ├─49396 /usr/bin/python /usr/bin/salt-master
	           ├─49399 /usr/bin/python /usr/bin/salt-master
	           └─49400 /usr/bin/python /usr/bin/salt-master

	Jun 27 19:19:26 localhost.localdomain systemd[1]: Starting The Salt Master Se...
	Jun 27 19:19:28 localhost.localdomain systemd[1]: Started The Salt Master Ser...
	Hint: Some lines were ellipsized, use -l to show in full.
```

**重启 minion**

```
	systemctl restart salt-minion.service
```

_补充：_

centOS-7 以前的版本重启服务的命令是`service restart xxxxxx(服务名)`

## step 4：master 同步所有主机

可以在同步前，先查看当前 salt-key 信息

```
	[root@localhost vimiix]# salt-key -L
	Accepted Keys:
	  local-minion
	Denied Keys:
	Unaccepted Keys:
	Rejected Keys:
```

同步所有主机

```
	salt-key -A
```

## step 5： 测试

**测试被控主机是否连通**

```
	[root@localhost vimiix]# salt "*" test.ping
	local-minion:
	    True
```

`local-minon` 就是刚刚我设置的客户端主机在服务器端的名称，连接正确，配置完成。

**远程命令测试**

```
	[root@localhost vimiix]# salt "*" cmd.run "uptime"
    local-minion:
     19:51:04 up  4:23,  2 users,  load average: 0.08, 0.06, 0.41
```

# SaltStack 执行命令的格式和 Python API

## 命令格式

> salt [argv] object command [argument]

分为以下几个部分：

- `salt` saltstack 的“发动机”
- `argv` 命令参数
- `object` 要执行命令的对象
- `command` 要执行的命令
- `argument` 要执行的命令的参数

**argv:**

`-E` 指定选择要执行命令的对象时用正则来匹配对象

```
	[root@localhost vimiix]# salt -E "^local+" test.ping
	local-minion:
	    True
```

`-L` 指定选择要执行命令的对象时采用列表的方式

```
	#我本机只连接了创建了一个主机，下面的‘*’可以替换为每个主机名，以“，”分隔每个主机名
	[root@localhost vimiix]# salt -L "local-minion,*" test.ping
	local-minion:
	    True
```

**command:**

`test.ping` 测试客户端是否连通

`cmd.run` 执行 linux 命令

```
	[root@localhost vimiix]# salt '*' cmd.run 'free -m'
	local-minion:
	                total      used       free      shared  buff/cache   available
	    Mem:          976       660        102           4         214         106
	    Swap:        2047        78       1969

```

## python API

在部署 saltstack 的同时本地 python 会具有一个 salt 的模块

示例：

```Python
	#!/usr/bin/python
	#coding:utf-8

	import salt.client

	client = salt.client.LocalClient()
	ret = client.cmd("*","test.ping")
	print(ret)
```
