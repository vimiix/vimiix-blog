---
title: 'Linux-git branch用法(查看、新建、删除、重命名)'
date: 2017-03-08 10:41:08
tags:
  - Linux
  - git
  - branch
categories: Linux
thumbnail: https://static.vimiix.com/uPic/2021-04-06/KQLlUb.jpg
---

![](https://static.vimiix.com/uPic/2021-04-06/KQLlUb.jpg)

<!--more-->

## 查看分支 git branch [-r|-a]:

1、git branch 查看本地所有分支

2、git branch -r 查看远程所有分支

3、git branch -a 查看本地和远程所有分支

## 新建分支 git branch [-f] <branchname>：

1、git branch <branchname> 新建分支，但不切换到新分支

2、git checkout -b <branchname> 新建并切换到新分支

## 删除分支 git branch (-d|-D) <branchname>:

1、git branch -d <branchname> 删除本地分支

2、git branch -d -r <branchname> 删除远程分支，删除后，还要推送（ git push origin :<branchname> ）到服务器才可以。

## 重命名分支 git branch (-m|-M) <oldbranch> <newbranch>:

使用-m 表示强制重命名。

如果需要重命名远程分支，建议的做法是：

1、删除远程待修改的分支

2、push 本地新分支名到远程

<hr>

## it 中 参数解释

### -d

    --delete:删除

### -D

    --delete --force 强制删除的快捷方式

### -f

    --force:强制

### -m

    --move:移动或重命名

### -M

    --move --force 强制移动或重命名的快捷方式

### -r

    --remote：远程

### -a

    --all:所有
