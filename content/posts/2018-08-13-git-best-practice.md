---
title: 'Git开发记录-合并多条commit最佳实践'
date: 2018-08-13 19:22:48
categories: 'git'
tags: ['git']
---

## 问题

常规的多人基于 GIT 协作开发的时候，都是遵循先 fork 一份主版本代码到自己的账号下面，然后基于本账户的版本，开分支来开发功能或修 Bug，完成以后再讲修改的内容，提交一个完整的 PR 贡献回主版本。

在本分支上开发的过程中，有时候不得不先提交到自己账号下面的克隆版本中来测试（比如豆瓣的`dae pre`，无法在本地生成预览，需要提交到远端），我们不能保证一次性提交就做到完美，避免不了会往复的修改后提交，这样的一次次测试用的 commit 属于是冗余的琐碎信息，对于主版本迭代是没有价值的。如果直接在基于该分支提交 PR，甚至被`merge`到`upstream/master`主版本中，这些不必要的 commit 信息也会包含进主版本中。这当然不是一个理想的迭代方式。

现在问题明确以后，就是一个目标：**将这些开发中的所有 commit 都合并为一条有意义的 commit 信息提交给主版本**。

<!--more-->

## 解决方案

为了实现这一目标，有两种方式，根据情况来选择：

### git rebase -i HEAD~n

`git log`查看当前分支，假设现在有三条 commit 记录，如下：

![image-20180813185617977](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/image-20180813185617977.png)

想合并最近这两天记录为一条，可以这样做：

```bash
git rebase -i HEAD~3
```

`HEAD` 就是 git 当前的版本指针，不懂的自行查阅。

`3` 代表要合并之前的几条记录

这时会出现下面这个编辑界面，你需要选择`pick`（选择）第一条来作为合并后提交的 commit 信息.

![image-20180813185149617](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/image-20180813185814408.png)

将要合并的条目前面的`pick`修改为 `s`或者 `squash`，都可以一个是缩写一个是全称（压缩的意思）git 只能将 log 向前压缩合并，所以选择最早的一条记录作为合并后的提交信息。

![image-20180813185847494](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/image-20180813185847494.png)

`:wq`保存退出，终端会出现类似这样的分支：

```bash
git:(0184399) ✗
```

`0184399` 这个是合并过程中的一个临时版本号，因为压缩是需要一条一条来合并的，这其中遇到冲突的话，需要手动处理以后才能合并。

如果没有冲突的话，执行下面的指令继续合并：

```bash
git rebase --continue
```

直到合并结束，终端的分支名会再次出现你自己创建的分支名就成功了。

如果过程中任何时候想放弃合并，执行下面的指令即可结束本次合并操作流程:

```bash
git rebase --abort
```

执行成功以后，再次`git log`查看，出现下面这样，就表示合并成功了，三条记录合并到一条了：

![image-20180813190354456](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/image-20180813190354456.png)

这时候，就可以推送到远端了，因为之前测试远端有我们提交的 commit 信息，此时和本地已经不一致了（因为我们的上述操作），所以简单的`git push`上去了，会被远端拒绝，这时候就需要霸道一点，使用`--force`参数强行覆盖远端的版本，就可以了。

```bash
git push --force
```

这种方案的好处是，也许我们之前就已经在主版本中提交了 PR，后面有添加了修改，合并后，PR 会自动更新 commit 信息，不需要重新开 PR，就可以 merge。但是比较繁琐的是需要一步一步手动处理自己在每次 commit 中的冲突

### git checkout -b

第二种方案比较简单，采用**开发-发布**分离的思想。开发使用一个分支，提交 PR 发布的时候，直接新开一个分支，将开发分支的更新后的代码 merge 过来即可。无需关心开发过程中经历了怎么样的“磨难”，只关心结果。

具体操作流程：

```bash
# 切换到主分支
$ git checkout master

# 从上游主版本更新代码到最新
$ git pull upstream master

# 新建一个发布分支
$ git checkout -b release_branch

# 使用带 `no-commit&squash` 参数的方式将开发分支的代码合并过来
# 这仿佛就好像你在开发分支上一次性完美地写完了所有代码
$ git merge origin/working_branch --no-commit --squash

# 添加commit 信息提交 OK
$ git commit -m "the work is done and ready for release"
```

是不是很帅气呢？反正我觉得是~~

可是如果提交了 PR 以后，code review 没通过，这时候，又进行了多次的修改以后，就需要关闭本次 PR，重新这样操作以后开辟新的 PR 来提交。如果 code review 过程中，可能有有价值的讨论信息在 PR 中，就需要使用第一种方案，来合并了。

—— EOF ——
