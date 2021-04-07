---
title: "Linux-软件包管理"
date: 2017-03-13 22:41:08
tags:
  - Linux
  - rpm
  - yum
categories: Linux


---

###### 来一首歌当背景音乐

<iframe frameborder="no" border="0" marginwidth="0" marginheight="0" width=330 height=86 src="//music.163.com/outchain/player?type=2&id=30854966&auto=0&height=66"></iframe>

<!--more-->

## rpm软件包管理

简称: Redhat Package Manager

##### 挂载光盘
[root@localhost ~]# umount /dev/sr0   卸载
[root@localhost ~]# mount /dev/sr0 /mnt/   挂载

##### rpm包名结构

zsh-5.0.2-14.el7.x86_64.rpm

|zsh|-5|.0|.2|-el7|x86|64|
|:----:|:----:|:---:|:----:|:--:|:--:|:--:|
|软件名|主版本号|次版本号|修订号|RHEL7|CPU架构平台|支持系统位数|

##### 安装rpm软件

- -i, --install           安装软件包
- --nodeps                不验证软件包依赖
- -v, --verbose           提供更多的详细信息输出
- -h, --hash              软件包安装的时候列出哈希标记

##### rpm查询功能

rpm -qa…

- -a  查询所有已安装的软件包
- -f  查询 文件所属软件包
- -p  查询软件包（通常用来看下还未安装的软件包）
- -i  显示软件包信息
- -l  显示软件包中的文件列表
- -d  显示被标注为文档的文件列表
- -c  显示被标注为配置文件的文件列表

通常可以配合管道 | more 来使用，使得结果更易读。

##### rpm包 升级

- rpm -Uvh [包名]

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/rpm-Uvh.png)

##### rpm包 卸载

- rpm -e [包名]

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/rpm-e.png)

## yum软件包管理

YUM 自动装软件包（软件包管理）

解决依赖关系问题、自动下载软件包，基于C/S架构。

##### 配置本地yum源的配置文件

	[root@localhost ~]#monut /dev/sr0 /mnt      #挂载光盘    
	[root@localhost ~]# rm -rf /etc/yum.repos.d/*
    [root@localhost ~]# vim /etc/yum.repos.d/rhel7.repo
    [rhel7-yum]					#yum源名称，唯一的，用来区分不同的yum源
    name=rhel7-source			#对yum源描述信息
    baseurl=file:///mnt			#yum源的路径（repodata目录所在的目录）
    #或baseurl=http://mirrors.aliyun.com/help/epel
    #或baseurl=ftp://192.168.1.63/pub
    enabled=1					#为1，表示启用yum源
    gpgcheck=0					#为1，使用公钥检验rpm的正确性


##### 主要操作

- 安装 yum  install    -y
- 检测升级 yum  check-update
- 升级 yum  update
- 软件包查询 yum  list
- 软件包信息 yum  info
- 卸载 yum  remove
- 帮助 yum  -help 或 man  yum

#### 安装一组软件包

- yum grouplist    查看包组
- yum groupinstall "Security Tools"
- yum groupinstall "安全性工具" -y

