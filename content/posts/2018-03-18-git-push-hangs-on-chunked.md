---
title: '[笔记]git push卡主不动问题记录：Git push hangs on POST git-receive-pack'
date: 2018-03-18 18:20:48
categories: 'git'
tags: ['git', 'note', 'solution']
---

## 问题

昨天完成了[《一个完整的 Django 入门指南》 - 第 6 部分](https://github.com/pythonzhichan/django-beginners-guide/blob/master/ClassBasedViews.md)的翻译工作，本地在翻译的过程中，存储了十几张原文中的 `png` 格式的插图。

在 `git push` 提交 **github** 仓库的时候，终端显示写成功 100%, 但是一直卡在了下面这里没有推送成功：

```bash
Counting objects: 21, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (21/21), done.
Writing objects: 100% (21/21), 1018.52 KiB | 17.87 MiB/s, done.
Total 21 (delta 7), reused 0 (delta 0)
# 卡在这里
```

<!--more-->

终止加上 `-v` 参数查看详细信息，显示：

```bash
POST git-receive-pack (chunked)
```

## 解决方法

在提交远程仓库的时候 git 在这里卡主了，经过搜索得知，这是 git 中的一个 bug ;在使用 https 传输时，如果上传的内容大小超过了一个预设上限值时，git 会使用分块编码的方式将内容上传。但这块没有实际奏效。

一个简单的解决办法是设置一个很大很大的上限值，这样让 git 不要对文件分块，就可以了。

设置这个值的方式有两种方法：

### 1、

在要提交的仓库目录下，输入下面的指令，其中 `52428800` 不是一个固定的值，可以根据你自己需求自己设定：<sup>[1]</sup>

```bash
git config http.postBuffer 524288000
```

### 2、

第二种方式是，打开项目目录下面的 **.git/** 文件夹下面的 **config** 文件， 在文件末添加 `postBuffer` 的值：

打开文件：

```bash
vim .git/config
```

追加参数：

```
# ...
[http]
     postBuffer = 524288000
```

## More

这两种方法最后实现的效果都是一样的，第一种方法是第二种方法的自动化写入配置参数的方式。

修改过以后，当再次执行 **git push** 的时候，你会发现，还是会卡在这里，但是不用着急，文件本身就比较大，耐心的等一会就可以成功提交了。

最后，其实最好的解决方法，我觉得就是不要在 github 上存储代码以外的大文件，大的文件可以上传到 CDN, 通过链接来调用。如果要存图片可以考虑将 **png** 转格式为 **jpg** 格式。

## 参考链接

1. [[stackoverflow] Hanging at “POST git-receive-pack (chunked)”
   ](https://stackoverflow.com/questions/10790232/hanging-at-post-git-receive-pack-chunked)
