---
title: 'Linux-rhel7.2操作系统安装'
date: 2017-03-08 23:00:31
tags:
  - Linux
  - rhel7.2
  - 操作系统
categories: Linux
---

## RHEL7

红帽公司发布的企业 Linux7 版本。

#### RHEL6 与 RHEL7 的关键区别：

1. 显著提升  Docker  的兼容性 （Docker 容器级虚拟化技术）

2. 默认文件系统从 EXT4 改为 XFS

3. 系统管理的进一步简化

4. 新版本内核 3.10 提供更多文件系统的支持

## RHEL7 的安装

**启动画面**

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%871.png)

界面说明：

** Install Red Hat Enterprise Linux 7.2 ** #安装 RHEL 7.2

** Test this media & install Red Hat Enterprise Linux 7.2** #测试安装文件并安装 RHEL 7.2

** Troubleshooting** #修复故障

一般选择第一项就可以了

_注：在 Trobleshooting 模式下，界面如下：_

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%872.png)

界面说明：

Install Red Hat Enterprise Linux 7.2 in basic graphics mod #基本图形化安装

Rescue a Red Hat Enterprise Linux system #修复系统

Run a memory test #运行内存测试系统

Boot from local drive #本地设备启动

Return to main menu #返回主菜单

这里选择第一项，安装 RHEL 7.1，回车

**进入下面的界面，按回车开始安装**

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%873.png)

**选择语言**：中文-简体中文（中国）  #正式生产服务器建议安装英文版本，单击继续按钮

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%874.png)

进入一站式安装界面，在此界面，只需把所有带"!"内容的感叹号全部消除，便可进行安装

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%876.png)

**时区选择**，设置完成，单击完成按钮

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%877.png)

**键盘选择**，单击 + 按钮，添加新的键盘布局方式，选中要添加的语言，然后单击添加即可，添加完成后，单击完成按钮

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%878.png)

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%879.png)

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8710.png)

**验证光盘完整性**，单击验证，验证光盘或镜像是否完整，防止安装过程出现软件包不完整

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8711.png)

**额外软件仓库**，可以不用选择，单击完成按钮

**软件包选择**，初学者建议选择带 GUI 的服务器，同时把开发工具相关的软件包也安装上，然后单击完成

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8713.png)

**选择-系统-安装位置**，进入磁盘分区界面

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8714.png)

选择-其它存储选项-分区-我要配置分区，点左上角的“完成”，进入下面的界面，在分区方案有标准分区，btrfs，LVM，LVM 简单配置，这里默认**LVM**就可以，然后单击创建新的分区，分区提前规划好，一般 swap 分区为物理内存的 1.5~2 倍（物理内存 2G 的话，swap 可以设置 4G），**/boot**分区 500M，**/**分区 10G，实际工作中可以创建数据分区，一般把数据和系统分开

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8715.png)

**创建/boot 分区**

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8716.png)

设备类型选择默认的标准分区，文件系统类型为 xfs，RHEL7 支持 brtfs，生产环境不建议选择，btrfs 文件系统目前技术尚未成熟，只是作为一种前瞻技术

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8717.png)

**创建 swap 分区**

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8718.png)

**创建/分区（根分区）**

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8719.png)

**必须创建的分区**

- /boot
- /
- swap 交换分区，虚拟内存

swap 创建标准：物理内存 1.5~2 倍
如果物理内存超过 4G，一般直接把 swap 创建 4G 就够了

Swap 分区的应用场景： 当物理内存不够用的时候 使用交换分区。

**必须与根目录放在同一分区的目录：**

- /bin
- /sbin
- /lib
- /etc
- /dev

这五个目录。绝对不可与/所在的分区分开，因为这五个目录，包含必要的系统工具与资料。当根目录开机被挂载进来时，需要这些工具与资料来维持正常的运作。若是把这五个目录放到其它分区中，系统就不能正常引导。

**可以与根目录放在不同分区的目录（不建议）：**

- /cdrom
- /mnt
- /media
- /proc
- /run
- /sys
- /srv 等

这些目录虽然可以放到其它的分区，但不需要这么做。因为这些目录只是为了维持运行所需，且大多不会占用空间，放到其它分区，也无益于系统的性能。如/mnt,/media, /cdrom 只是为实体存储媒体提供挂载点而已；又如/proc，/sys 其实是内存上的数据，上面所有的数据完全不会占用硬盘的空间。所以这些目录不需要额外的分区存放。

分区创建完成，单击完成按钮，弹出下图，点击**接受更改**

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8720.png)

**KDUMP 选项配置**，本选项普通用户去掉勾选，关闭即可

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8721.png)

**网络配置**，开启以太网连接，将会自动获取 IP 地址，如果要手动配置，单击“配置(O)…”按钮

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8722.png)

全部配置完成之后，单击开始安装，进行系统安装

进入安装界面，这里需要配置用户密码

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8723.png)

**安装过程**（然后就是漫长等待...可以去抽根烟了）

**进入启动界面**

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8724.png)

**首次启动配置，许可认证->同意许可->完成配置**

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8725.png)

**登录系统**，选择未列出，输入用户名 root

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/%E5%9B%BE%E7%89%8726.png)

**搞定！**
