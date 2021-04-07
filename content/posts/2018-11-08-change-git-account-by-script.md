---
title: '快速切换本地Git用户记录'
date: 2018-11-08 23:22:48
categories: 'git'
tags: ['git', 'tool', 'bash']
---

现在大部分的科技公司开发模式，都是基于 Git 进行多人协作开发。所以，对于我们每一个开发者来说，Git 的操作就是必不可少的技能了（不是锦上添花，而是必不可少）。对于 Git 的操作，不是本次记录的内容，网上的[教程](https://git-scm.com/book/zh/v1/%E8%B5%B7%E6%AD%A5-Git-%E5%9F%BA%E7%A1%80)可以在官网找到。

今天我想记录一下我本机多用户管理的一点小操作。

当每进入一家新公司的时候，总会在新公司领到一个新的公司邮箱，基本上这个邮箱也就是你在公司期间进行代码开发的 git 账户。这时候，加上我们平时在 [GitHub](https://github.com) 的账户，就会有两个账户需要切换使用。

下面是我个人的一点小技巧记录，不一定是最好的，但只要自己用着方便就 OK，如果此时看文章的你有好的方法的话，可以请在讨论区交流。

<!--more-->

### 脚本

核心思想就是把人需要使用的指令，用 bash 脚本代替，然后加入到环境变量。

下面是我的脚本内容：

- change_git_account.sh

```bash
#!/bin/bash

if [ -z $1 ];then
    echo "至少输入一个账户名"
elif [ $1 = vimiix ];then
    echo "切换账户为 $1 ..."
    git config --global user.email "vimiix@yyy.com"
    git config --global user.name "vimiix"
    echo "切换成功"
elif [ $1 = yaoqian ];then
    echo "切换账户为 $1 ..."
    git config --global user.email "yaoqian@xxx.com"
    git config --global user.name "yaoqian"
    echo "切换成功"
else
    echo "请输入本机器内正确的用户名，坏人~~~"
fi

```

使用脚本的方法就是：

```bash
bash change_git_account.sh [用户名]
```

在我这里，用户名有两个选项 `vimix`和`yaoqian` ，当不加参数或输入其他参数的时候都不会切换。

### 设置别名

为了更方便的使用这个脚本，我把他加入到了 profile 中，设置了别名，以后就可以直接通过别名全局使用了。

我使用的是 `iTerm +OhMyZsh`，就打开 `~/.zshrc` 文件在最后加入下面的语句（注意脚本的路径根据脚本的路径配置），如果你是 `bash` 就在`~/.bashrc` 中追加。

```bash
alias chgit-user="bash /Users/vimiix/Work/Tools/change_git_account.sh"
```

关闭文件后，在当前终端执行 `source` 来让指令生效(或者打开一个新的终端 tab 即可)：

```bash
source ~/.zshrc
```

### 使用自定义指令

经过上面的配置，我们自定义的指令就生效了，可以在新的终端中尝试一下效果：

```bash
➜ vimiix ~ chgit-user vimiix
切换账户为 vimiix ...
切换成功
➜ vimiix ~ chgit-user yaoqian
切换账户为 yaoqian ...
切换成功
➜ vimiix ~ chgit-user
至少输入一个账户名
➜ vimiix ~ chgit-user xxx
请输入本机器内正确的用户名，坏人~~~
➜ vimiix ~
```

--- EOF ---
