---
title: '[Python]使用pipenv管理python虚拟环境'
date: 2018-03-11 18:20:48
categories: 'Python'
tags: ['pipenv', 'autoenv', 'Python', 'virtualenv']
---

## 前言

近期的项目中，我开始尝试着从 virtualenv 管理 python 虚拟环境，切换到用 pipenv 来管理。

经过一段时间的使用，着实觉得 pipenv 使用的更加顺手，更加的便捷。这当然也延续了 [Kenneth](https://www.kennethreitz.org/) 大神一贯的项目作风-- **`For Humans`** 。

再配合上 **`autoenv`**（自动激活虚拟环境的工具），更加的 Perfect！

<!--more-->

## pipenv + autoenv 的使用

### 安装

我们通过`pip`即可快速安装

```bash
pip install autoenv  # 安装 autoenv
pip install pipenv #  安装 pipenv
```

_注：这里简单略过安装部分，网上大把的教程有教如何在各个系统环境下安装，不是本文的主要内容_

### pipenv 的使用

_前提：假设我们在用户目录 `~` 下有一个项目叫 `my_project/`_

我们首先进入项目目录：

```
cd  ~/my_project
```

进入项目以后，如果直接执行 `pipenv install`，`pipenv` 会根据系统默认的 python 版本，来创建虚拟环境。（前提是本项目中不存在已有的`Pipfile`，如果有，它会去根据`Pipfile`安装对应的版本和`Pipfile`中记录的依赖库）。

但是我们一般会创建虚拟环境的时候指定 python 版本,就需要配上 `--two`或者`--three` 这个参数:

- --two 使用 python2 来创建虚拟环境
- --three 使用 python3 来创建虚拟环境

```bash
pipenv install --three  # 需要确保系统中存在python3版本
```

下面是执行后的输出信息：

```
Creating a virtualenv for this project…
Using /usr/local/bin/python3 to create virtualenv…
# 这里可以看到继承自哪个python版本
⠋Running virtualenv with interpreter /usr/local/bin/python3 Using base prefix '/usr/local'
New python executable in /root/.local/share/virtualenvs/my_project-dhpIKgdN/bin/python3
Also creating executable in /root/.local/share/virtualenvs/my_project-dhpIKgdN/bin/python
Installing setuptools, pip, wheel...done.

# 这里可以看到虚拟环境安装的位置
Virtualenv location: /root/.local/share/virtualenvs/my_project-dhpIKgdN
Creating a Pipfile for this project…
Pipfile.lock not found, creating…
Locking [dev-packages] dependencies…
Locking [packages] dependencies…
Updated Pipfile.lock (625834)!
Installing dependencies from Pipfile.lock (625834)…
  🐍   ▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉ 0/0 — 00:00:00
To activate this project's virtualenv, run the following:
 $ pipenv shell # 要激活该虚拟环境，执行这条指令
```

这将在项目目录中创建两个新文件 `Pipfile` 和 `Pipfile.lock`

- Pipfile 存放着当前虚拟环境的配置信息，包含 python 版本，pypi 源，以及项目安装的依赖库。 - `pipenv`根据这个来寻找项目的根目录。 - `Pipfile` 文件是 `TOML` 格式而不是 `requirements.txt` 那样的纯文本。<sup>[1]</sup> - 一个项目对应一个 Pipfile，支持开发环境与正式环境区分。默认提供 `default` 和 `development` 区分
- Pipfile.lock 顾名思义，这个文件时对于`Pipfile`的一个锁定。支持锁定项目不同的版本所依赖的环境.

激活虚拟环境, 在当前目录执行下面的指令即可：

```bash
pipenv shell
```

#### 创建 .env 文件

如果你和我一样想要使用 `autoenv` 来自动激活虚拟环境，就在当前项目的根目录下面，创建一个名为 **.env** 的文件，将 `pipenv shell` 写进去。具体使用在后面说明。

如果没有这个需求，可以忽略这一段话。

### 更新 pypi 源来提高依赖库安装的速度

在使用 pipenv 的时候，常常会在安装的时候，一直卡在了 `Locking` 这里，通过加上 `-v` 参数，可以看到安装过程中的步骤信息，卡在了下载那里，这时应该可以意识到是因为网络的原因，pipenv 创建的 `Pipfile` 中默认的 pypi 源是 python 官方的 **https://pypi.python.org/simple**。我们国内用户访问下载的时候会很慢。

所以，我一般会在创建好 Pipfile 以后，修改到文件中 `source` 块下的 `url` 字段，设置为国内的 pypi 源就好了，我推荐的是清华的 pypi 源，具体设置如下：

###### _备注：我还没有找到如何修改能在创建时就设好的方法，应该修改源码是可以的，但这样不尊重源码，毕竟高墙是我们自己筑起的，如果有好的方法，您不妨在评论中告诉我一下_

```
[[source]]

 url = "https://pypi.tuna.tsinghua.edu.cn/simple"
 verify_ssl = true
 name = "pypi"

```

### 管理 Python 依赖关系

`Pipfile` 包含关于项目的依赖包的信息，并取代通常在 Python 项目中使用的 `requirements.txt` 文件。 如果你之前的项目中存在`requirements.txt`文件，`pipenv` 可以很轻松的安装 `requirements.txt` 中的依赖包。

```bash
pipenv install -r requirements.txt
# 或者
pipenv install --requirements requirements.txt
```

可以通过更新 `Pipfile.lock` 来冻结软件包名称及其版本以及其自己的依赖关系的列表。 这时需要使用`lock`关键字来完成，

```
pipenv lock
```

如果我们想要在虚拟环境中安装某个指定的库，比如 `requests`, 直接在 install 后面跟上就可以了:

```
pipenv install requests
```

如果想查看当前环境中第三方包之间的依赖关系,可以通过 `pipenv graph` 来查看：

```
[root@VM_27_243_centos my_project]# pipenv graph
requests==2.18.4
  - certifi [required: >=2017.4.17, installed: 2018.1.18]
  - chardet [required: <3.1.0,>=3.0.2, installed: 3.0.4]
  - idna [required: <2.7,>=2.5, installed: 2.6]
  - urllib3 [required: <1.23,>=1.21.1, installed: 1.22]

```

从输出可以看出，我们按照的 `requests` 包，依赖于其他的四个包，pipenv 帮你自动管理着这些包这件的依赖关系、。我们可以看到 `requests` 依赖于 `urllib3`， 假设我们再安装一个包，并且这个包也同样依赖着 `urllib3` ,当我们要卸载掉 `requests` 的时候，pipenv 会自动检测这些包之间的依赖关系，因为 `urllib3` 依旧有其他包依赖，所以会保留，只会卸载掉其他的依赖库。（_卸载的指令是`pipenv uninstall`_）

### 退出虚拟环境

任何时候想退出虚拟环境，只需一条简单的 `exit` 指令即可

```
exit
```

### autoenv 的使用

安装好 `autoenv` 以后，`autoenv` 可以在进入项目之后自动检测项目目录的 `.env` 文件激活项目所需的虚拟环境，这样能够保证每次切换不同项目的时候，都能自动进入相应项目所依赖的虚拟环境。

要实现 `autoenv` 自动识别项目目录中的 `.env` 文件，需要将 `autoenv` 的激活脚本添加到终端的 `profile` 中：

```bash
# bash 的话，执行这一条指令
echo "source `which activate.sh`" >> ~/.bashrc
# zsh 的话，执行这一条指令
echo "source `which activate.sh`" >> ~/.zshrc
```

这样就配置好了 `autoenv` 了，当 `cd` 到项目目录中后，就会自动激活虚拟环境了，如果是第一次，系统会提示你确认是否以后都自动激活，输入 `y` ，然后回车确认即可。

## <sup>\*</sup> pipenv 详细参数和指令说明<sup>[2]</sup><sup>[3]</sup>

### pipenv

```
pipenv [OPTIONS] COMMAND [ARGS]...
```

**Options**：

- **--upgrade**

      	更新pipenv＆pip到最新版本

- **--where**

      	输出项目根目录信息

- **--venv**

      	输出虚拟环境信息

- **--py**

      	输出python解释器的信息

- **--envs**

      	输出环境变量信息

- **--rm**

      	删除当前虚拟环境

- **--bare**

      	精简输出

- **--man**

      	显示帮助手册页面

- **--three, --two**

      	使用python3/2创建虚拟环境

- **--python** \<python\>

      	指定应该使用哪个版本的Python虚拟环境

- **--site-packages**

      	为虚拟环境启用site-packages

- **--version**

      	显示版本并退出

### check

检查装的包的安全性

```
pipenv check [OPTIONS] [ARGS]...
```

**Options**：

- **--three, --two**

      	使用python3/2创建虚拟环境

- **--python** \<python\>
  指定应该使用哪个版本的 Python 虚拟环境
- **--system**

      	使用系统的Python

- **--unused** \<unused\>

      	指定项目路径，显示可能未使用的依赖关系。

### clean

卸载未在 pipfile.lock 中指定的所有软件包

```
pipenv clean [OPTIONS]
```

**Options**:

- **-v，--verbose**
  详细信息模式
- **--three, --two**

      	使用python3/2创建虚拟环境

- **--python** \<python\>
  指定应该使用哪个版本的 Python 虚拟环境
- **--dry-run**

      	只输出不需要的包

### graph

显示当前安装的依赖关系图信息

```
pipenv graph [OPTIONS]
```

**Options**:

- **--bare**

      	最小的输出 *(备注：这个不太理解，尝试过指定了这个参数会输出所有的包，不指定却只输出pip安装的包，感觉和官方注释相反)*

- **--json**

      	输出JSON格式

- **--reverse**

      	逆转依赖关系图

### install

安装参数提供的软件包并将它们添加到**pipfile**，如果没有提供参数，就安装所有软件包。

```
pipenv install [OPTIONS] [PACKAGE_NAME] [MORE_PACKAGES]...
```

**Options**:

- **-d, --dev**

      	在[dev-packages]中安装软件包

- **--three, --two**
  使用 python3/2 创建虚拟环境
- **--python** \<python\>

      	指定应该使用哪个版本的Python虚拟环境

- **--system**

      	系统pip管理

- **-r, --requirements** \<requirements\>

      	导入一个requirements.txt文件。

- **-c, --code** \<code\>

      	从代码库导入

- **-v, --verbose**

      	详细信息模式

- **--ignore-pipfile**

      	在安装时忽略pipfile，使用pipfile.lock

- **--sequential**

      	一次只安装一个依赖项，而不是同时安装。

- **--skip-lock**

      	相反，在安装时忽略锁定机制 - 使用pipfile

- **--deploy**

      	如果pipfile.lock过时或python版本错误则中止

- **--pre**

      	允许预发布

- **--keep-outdated**

      	防止在pipfile.lock中更新过时的依赖关系

- **--selective-upgrade**

      	更新指定的包

**Arguments**:

- **PACKAGE_NAME**
  包名（可选参数）
- **MORE_PACKAGES**

      	多个包（可选参数）

### lock

生成**pipfile.lock**

```
pipenv lock [OPTIONS]
```

**Options**:

- **-v，--verbose**
  详细信息模式
- **--three, --two**

      	使用python3/2创建虚拟环境

- **--python** \<python\>
  指定应该使用哪个版本的 Python 虚拟环境
- **-r, --requirements**

      	生成与requirements.txt兼容的lock文件

- **-d, --dev**

      	生成与requirements.txt兼容的开发模式依赖项

- **--clear**

      	清除依赖关系缓存

- **--pre**

      	允许预发布

- **--keep-outdated**

      	防止在pipfile.lock中更新过时的依赖关系

### open

在编辑器中查看指定的模块

```
pipenv open [OPTIONS] MODULE
```

**Options**：

- **--three, --two**

      	使用python3/2创建虚拟环境

- **--python** \<python\>
  指定应该使用哪个版本的 Python 虚拟环境

**MODULE**

模块名（必填项）

### run

运行一个虚拟环境中的命令(就是在未激活虚拟环境时可以直接用虚拟环境的 python 执行）

```
pipenv run [OPTIONS] COMMAND [ARGS]...
```

**Options**:

- **--three, --two**

      	使用python3/2创建虚拟环境

- **--python** \<python\>
  指定应该使用哪个版本的 Python 虚拟环境

**Arguments**

- **COMMAND**

      	命令（必填项）

- **ARGS**

      	参数（可选项）

### shell

在虚拟环境中产生一个 shell(就是激活虚拟环境)

```
pipenv shell [OPTIONS] [SHELL_ARGS]...
```

**Options**:

- **--three, --two**

      	使用python3/2创建虚拟环境

- **--python** \<python\>
  指定应该使用哪个版本的 Python 虚拟环境
- **--fancy**

      	以好看的模式运行shell（用于如果优化过配置的shell）

- **--anyway**

      	产生一个子shell，即使已经在虚拟环境中

      	*这个比较很少用，但就是虚拟环境中的虚拟环境，执行后在终端前面看到两个虚拟环境的括号：*

      	```
      	(my_project-dhpIKgdN) (my_project-dhpIKgdN) [root@centos my_project]#
      	```

**Arguments**:

- **SHELL_ARGS**
  可选项

### sync

安装所有在 pipfile.lock 中指定的软件包

```
pipenv sync [OPTIONS]
```

**Options**:

- **-v，--verbose**
  详细信息模式
- **-d, --dev**

      	另外在[dev-packages]中安装软件包

- **--three, --two**

      	使用python3/2创建虚拟环境

- **--python** \<python\>
  指定应该使用哪个版本的 Python 虚拟环境
- **--bare**

      	精简输出

- **--clear**

      	清除依赖关系缓存

- **--sequential**

      	一次只安装一个依赖项，而不是同时安装

### uninstall

卸载指定的软件包并将其从**pipfile**中删除

```
pipenv uninstall [OPTIONS] [PACKAGE_NAME] [MORE_PACKAGES]...
```

**Options**:

- **-v，--verbose**
  详细信息模式
- **--three, --two**

      	使用python3/2创建虚拟环境

- **--python** \<python\>
  指定应该使用哪个版本的 Python 虚拟环境
- **--system**

      	系统的pip管理

- **--lock**
  卸载之后锁定
- **--all-dev**

      	从[dev-packages]中卸载所有软件包

- **--all**

      	从虚拟环境中清除所有包。但不会编辑pipfile

- **--keep-outdated**

      	防止在pipfile.lock中更新过时的依赖关系

**Arguments**:

- **PACKAGE_NAME**
  包名（可选参数）
- **MORE_PACKAGES**

      	多个包（可选参数）

### update

更新指定包

```
pipenv update [OPTIONS] [PACKAGES]...
```

**Options**:

- **-d, --dev**

      	在[dev-packages]中安装软件包

- **--three, --two**
  使用 python3/2 创建虚拟环境
- **--python** \<python\>

      	指定应该使用哪个版本的Python虚拟环境

- **-v, --verbose**

      	详细信息模式

- **--clear**

      	清除依赖关系缓存

- **--bare**

      	精简输出

- **--pre**

      	允许预发布

- **--keep-outdated**

      	防止在pipfile.lock中更新过时的依赖关系

- **--sequential**

      	一次只安装一个依赖项，而不是同时安装

- **--outdated**

      	列出过时的依赖关系

- **--dry-run**

      	列出过时的依赖关系

### 参考链接：

[1] : [使用 pipenv 管理你的项目 -- 董伟明](http://www.dongwm.com/archives/%E4%BD%BF%E7%94%A8pipenv%E7%AE%A1%E7%90%86%E4%BD%A0%E7%9A%84%E9%A1%B9%E7%9B%AE/)

[2] :[Pipenv: Python Dev Workflow for Humans](https://docs.pipenv.org/)

[3] :[https://pypi.python.org/pypi/pipenv](https://pypi.python.org/pypi/pipenv)
