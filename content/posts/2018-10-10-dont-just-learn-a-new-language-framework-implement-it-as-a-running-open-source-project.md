---
title: '[译]实践出真知'
date: 2018-10-10 23:22:48
categories: 'translation'
tags: ['note', 'translation']
---

Aristotle（亚里士多德），希腊著名的哲学家和科学家，曾经说过：“对于那些我们在能做到之前必须学习如何做的事情，我们需要边做边学 (_For the things we have to learn before we can do them, we learn by doing them._)”。想象一下，假如你已经读过 3 本关于骑行的书了，然后有人给你一辆自行车并让你骑它，你能骑吗？很显然，答案是“不能”。这无关乎你曾经读了多少关于骑行的书或你看了多少相关视频的事情。它需要你真正骑上一辆自行车，去保持平衡，去学习脚，手和眼睛的协调配合才能掌握的一件事情。学习新技术，新语言或框架同样也是如此。

如果现在你在想：我都不会某个语言或某个框架，我要怎么去实现这个开源项目呢，那么先停止抱怨。也许你是技术或编程的新手，但你需要知道如何去学习新事物。学习新东西的最好方法就是实践。这篇文章将重点关注普适通用的方法。它会帮助你从我想要学习'X'到我有一个项目在'X'运行，所以继续往下看。如果你决定通过做一个项目来学习新东西，那就把它开源吧。Github 是托管你的开源项目的首选服务商。在其上你可以享受很多的免费服务。这篇文章的编写主要面向编程起步者，但对于经验丰富的软件工程师也同样有用。

<!--more-->

### 摘要（[TLDR;](http://www.learnenglishwithwill.com/tldr-meaning-demystified/)）

> 通过编写项目来学习语言/框架，然后使用免费使用服务开源出去。不要只看课程，阅读文档，然后找到解决方案就完事，实践出真知。在项目中去使用 git 并尽量 docker 化。代码需要添加正确的代码质量检查服务以找到最佳实践，将项目部署到服务器上，让其可以对外通过 URL 访问。

### 不要只看课程，阅读文档，找到解决方案就完事

如今，学习新知识有很多选择。视频课程仍然是最受欢迎的媒介之一。你可以在[Udemy](https://www.udemy.com/)，[Pluralsight](https://www.pluralsight.com/)甚至[Youtube](https://youtube.com/)上学习。在你边做边学之前，观看视频只会在某种程度上有所帮助，更好的方式是阅读官方手册。例如，阅读 React JS 文档比仅通过观看 React JS 课程更好。这样你会发现创作者的思想在其中。理清创作一个 Javascript 框架/库背后的逻辑会帮助你找到最合适的解决方案。

### 学习使用 Git 进行协作

[“没有谁是一座孤岛”](https://www.douban.com/group/topic/32883040/) ，特别是在技术方面，你通常不会单独工作，肯定是作为团队的一员。因此，即使在学习新内容时，也要尝试找可以一起合作的人。与其它[流行的代码协作工具](https://trends.google.com/trends/explore?q=git,svn,mercurial,bazaar)相比，Git 非常受欢迎。当有超过 1 人为项目编写代码时，它是很有用的。你应该通过实践学习 git，可以通过查看[Github 教程](https://try.github.io/)。我强烈推荐边做边学。当你将代码推送到 Github 之后，任何人都可以为你的项目贡献力量。

### 实现 docker 化，克服我的机器综合症

都[8102 年](https://baike.baidu.com/item/8102%E5%B9%B4)了，如果你想让你的应用程序更易于使用，请使用[Docker](https://www.docker.com/)。这对于增加对开源项目的贡献也有很大帮助。使用`docker compose`在本地运行项目就像执行 2 个命令一样。Docker 有很多优点。对于初学者来说，这是一种确保你的应用可以在其他人的机器上以相同的方式运行的方法。在你部署应用程序的服务器上也是如此。只要它在 Docker 上运行良好，你就大可以放心，它保证可以在任何环境中无问题地运行。

### 添加代码质量检查

只是让你的项目能够运行不应该是你的最终目标。代码质量，稳定性等也应该是你追求的一个指标。你需要给你准备开源的项目添加代码质量检查流程（单元测试是必不可少的）。根据语言/框架，你可以选择任何服务。我推荐[Code Climate](https://codeclimate.com/quality/)。Code Climate 支持各种语言，从 Javascript 到 PHP，从 Java / Kotlin 到 Swift，适用于移动开发人员。通过最新的[浏览器插件](https://codeclimate.com/browser-extension/)，你可以在 Github 的 PR 界面直观的查看关于代码的检查情况。只需将其连接一次到你的 Github 仓库，就可以开始查看的你的代码质量报告。然后你就可以通过这个来改良你的代码。[举个例子](https://codeclimate.com/github/geshan/currency-api/src/exchangeRates.js/source)。

### 部署你的项目

设想现在你正在编写一个新项目来学习你喜欢的'X'语言或'Y'框架。你已经写了一部分，也有用 Git 协作，并在 Github 上开源了代码。你使用了 Docker，并且每次推送都会运行代码质量检查，非常棒！但你能把它展示给生活在不同城市/国家的朋友吗？当然可以！

你可以使用不同的服务来部署 Web 应用程序。通过 URL，你可以将其展示给你的朋友或知道 URL 的任何人。你可以免费部署到[Heroku](https://www.heroku.com/)或[Zeit Now](https://zeit.co/now)等服务。如果你有 Docker 化你的开源应用程序，我会推荐 Zeit Now。通过和 Github 集成，Zeit Now 将为每次 PR 提供一个新 URL。这使得测试变得轻而易举。你可以查看我[编写的演示货币转换器 API 应用程序的示例](https://github.com/geshan/currency-api/pull/9)。

### 总结

总而言之，边做边学是学习新事物的最佳方式。你的目标应该不仅是让它工作起来，而且还要遵循最佳实践。这就是代码质量发挥作用的地方。如果你可以添加自动化测试和持续集成，对于初学者来说，这一点属于稍微有点进阶的内容。祝你在实践中有所收获。

- 原文首发于：[https://geshan.com.np/blog/2018/10/dont-just-learn-a-new-language-slash-framework/](https://geshan.com.np/blog/2018/10/dont-just-learn-a-new-language-slash-framework/)

--- EOF ---
