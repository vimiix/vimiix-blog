---
title: 'Linux|CentOS系统yum安装pip及遇到的问题解决方法'
date: 2017-08-27 14:16:25
tags:
  - CentOS
  - Python
  - pip
categories: 'Linux'
---

**pip 是一个以 Python 计算机程序语言写成的软件包管理系统，他可以安装和管理软件包，另外不少的软件包也可以在“Python 软件包索引”（英语：Python Package Index，简称 PyPI）中找到。**

<!--more-->

## 常规的安装 pip 方法

首先安装扩展源 EPEL

```Bash
	sudo yum -y install epel-release
```

然后再安装 pip

```Bash
	sudo yum -y install python-pip
```

## 手动下载编译安装 pip

一般上面的方法就可以安装成功了。今天我在新买的云主机上面也这样做，epel 扩展源确认已经安装，但还是一直提示我 python-pip 这个模块找不到。这种情况，只能自己手动下载源文件编译安装了。

截至书写本文时间，pip 的最新版本为`9.0.1`

```Bash
	wget --no-check-certificate https://github.com/pypa/pip/archive/9.0.1.tar.gz
```

_注意：wget 获取 https 的时候需要加上：`--no-check-certificate`_

下载下来以后，解压安装

```Bash
	tar zvxf 9.0.1.tar.gz
	cd pip-9.0.1/
	python setup.py install
```

#### 缺少`setiptools模块`

此时在安装 pip 的过程执行`python setup.py install`时报错，提示：`ImportError No module named setuptools`字面意思就是本地缺少 setuptools 这个模块。

#### 解决方法

```Bash
	wget http://pypi.python.org/packages/source/s/setuptools/setuptools-0.6c11.tar.gz
	tar zxvf setuptools-0.6c11.tar.gz
	cd setuptools-0.6c11
	python setup.py install
```

`setuptools`安装成功以后，再返回去 pip 的解压目录执行`python setup.py install`，此时没有报错，且安装成功。

## pip 升级

pip 安装成功以后，我喜欢把软件都更新到最新版，更新 pip 执行

```Bash
	pip install --upgrade pip
```

至此，pip 就安装完整的安装成功了。

### _参考资料_

1. [pip（软件包管理系统）-维基百科](<https://zh.wikipedia.org/wiki/Pip_(%E8%BB%9F%E4%BB%B6%E5%8C%85%E7%AE%A1%E7%90%86%E7%B3%BB%E7%B5%B1)>)
2. [ pip 安装使用 ImportError: No module named setuptools 解决方法](http://blog.csdn.net/yangbodong22011/article/details/52456581)
3. [包子博客](http://www.jincon.com/archives/159/)
