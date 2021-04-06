---
title: 'Python|使用Anaconda完成Python多环境配置'
date: 2017-06-01 09:53:54
tags:
  - Python
  - Anaconda
categories: Python
---

今天是六月一号，国际儿童节，首先祝生活中的每一个宝宝（_只要你觉得自己是宝宝，那你就是_）<H3>Happy Children's Day!!!</h3>

OK,言归正传，昨天开始学习 Django 开发，个人认为，作为主流趋势，我倾向于用 pyhton3.x 开发，因为 web 开发有别与其他，对于实时响应要求相对较高，python3 可以更好的优化。在刚刚结束的 Pycon2017 上，来自 Instagram 的 Lisa Guo 和 Hui Ding（两位华裔）分别介绍了[Instagram 为何选择 py3 以及整个网站迁移 py3 的过程](https://www.youtube.com/watch?v=66XoCk79kjM)。虽然还不是很具体了解其中的差别，但一个体量不小的公司做出了向 python3 迁移的举动，一定说明 python3 在 web 开发上有肯定的优势。

<!--more-->

诚然，对于 devops 来说，使用 python2 的确是比 python3 更方便，虽然很多的主流框架还不支持 python3，但我觉得，这只是个时间问题。

因为我电脑上装的是 python2.7 版本，但学习 Django,又想用 py3,所以就面临了多版本开发的情况。经过一番搜索安装学习，我放弃了[virtualenv](https://virtualenv.pypa.io/en/stable/)，选择了用[Anaconda](https://www.continuum.io/downloads)，莫名的感觉这个略屌。

> windows 下用 python 非常的麻烦。所以想要一个包管理的东西，那么 Anaconda 是非常好的一个管理工具，无论你是想用 python2.7 还是 python3.4。

这句话是从网上摘的，主要是要强调一点:

> 对于 Anaconda 来说，任何模块都看作是一个包，甚至 python，甚至 anaconda 自己。

**接下来就开始 Anaconda 的发现之旅！**

### #0

首先需要下载 Anaconda,可以去[官网](https://www.continuum.io/downloads)选择你对应的系统版本下载。官网服务器在国外，如果下载速度太慢，清华镜像站也提供了下载地址([https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/](https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/))，至于 anaconda 的版本选择 2 还是 3，网上一致的说法是----**随便**，因为他是一个多版本管理工具，后面会创建不同 python 版本的分支。

### #1

然后就是安装，下载好以后双击开始安装，基本都直接选择下一步，文件安装路径可以自行选择一下。然后就是等待完成。

### #2

安装结束以后，打开`cmd`，输入`conda --version`查看版本号，如果显示出版本号，证明环境安装成功了。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/conda-version.png)

### #3

开始创建 python 多版本环境，这里先理解一个简单的概念，其实一个 python 环境，就是使用命令调用当前目录下的 python 编译器。不同的版本，可以理解为在不同文件夹下的不同 python 版本的编译器。没有创建分支环境时，anaconda 有个默认的分支`root`，这里不是根的意思，这个 root 指得就是系统环境的 python 环境。

**创建一个除了 root 分支之外的 2.7.× 的 python 环境**

```
	# 创建一个名为python27的环境，指定Python版本是2.7（不用管是2.7.x，conda会为我们自动寻找2.7.x中的最新版本）
	conda create --name python27 python=2.7
```

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/anaconda.png)

等待自动安装完成，到对应的目录下`（[Anaconda的安装目录]\env)`查看，自动生成一个 python27 的文件夹，就说明安装好了一个 python2.7 的环境了。

同理，在创建一个 python3.4 的环境

```
	conda create --name python34 python=3.4
```

再看目录，自动生成 python34 文件夹，那么就成功安装了两个版本的 python 环境。

**查看当前版本分支**

```
	conda info -e
```

在这里可以看到你所在的 python 环境分支（分支前面带个\*号），以及已安装的所有版本分支。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/anaconda-info.png)

**切换到需要的 python 版本分支**

```Python
	#window系统
	activate pyhton27

	#linux,OS X系统
	source activate python27
```

window 下直接在 cmd 里输入`activate python27`就可以切换到 python2.7 版本的环境下，终端在文件路径前多了一个`(python27)`,就表示切换成功了。

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/anaconda-activate.png)

进入以后就是和系统默认隔离的一个 python 环境，可以在这个环境里面肆意的造了，想装什么包，就装什么包，方法类似`pip`

```Pyhton
	#查找beautifulsoup4的包
	conda search beautifulsoup4

	#为python34安装beautifulsoup
	#NOTE: You must tell conda the name of the environment (--name bunnies) OR it will install in the current environment.你必须告诉conda你要安装包的环境的名称，不然会安装在当前环境下。我这里的环境就是python34
	conda install [--name python34] beautifulsoup4

	#查看你安装的包
	conda list
```

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/conda-install.png)

**退出当前 python 分支**

```
	#windows
	deactivate

	#linux, OS X
	source deactivate
```

我的 windows 系统，在当前环境下，输入`deactivate`，就退出了。

###### 更多

更多 conda 的指令，需查阅[官方手册](https://conda.io/docs/announcements.html)。
