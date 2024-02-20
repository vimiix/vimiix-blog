---
title: '[译]做一个拥有 Git 好习惯的开发者'
date: 2024-02-20 00:50:48
categories: 'git'
tags:
  - 'translation'
  - 'git'
  - 'best-practices'
---

如果你是一名开发人员，你可能每天都会使用 git 作为版本控制系统。这个工具的使用对于应用程序的开发过程是至关重要的，无论是在团队协作还是单独工作。但是，常常会遇到混乱的项目库，提交的 commit 信息不明确，不能传达有用的内容，以及滥用分支等问题。了解如何正确使用 git 并遵循良好的实践对于那些想要在就业市场中脱颖而出的人来说是必不可少的。

## Git 分支的命名约定

当我们在处理代码版本控制时，我们应该遵循的一个好的习惯，就是为分支、提交、拉取请求等使用**清晰**和**描述性**的名称。确保所有团队成员都有一个简洁的工作流程是至关重要的。除了提高工作效率之外，记录项目的开发过程也简化了团队合作。通过遵循这些实践，你很快就会获益其中。

基于此，开发者社区创建了一些分支命名的约定，你可以在项目中遵循这些约定。虽然遵循下列规则不是硬性的要求，但它们可以帮助你提高开发技能。

1. **使用小写字母**：分支名称不要用大写字母，强制小写;
2. **连字符分隔**：如果分支名称包含多个单词，请使用连字符将它们分开，遵循短横线命名法。避免使用帕斯卡命名法、驼峰命名法或蛇形命名法;
3. **(a-z, 0-9)**：分支名称中只使用字母数字字符和连字符，避免使用其他字符;
4. **不要使用连续的连字符(--)**：这种做法可能令人困惑。例如，如果你有分支类型(如feature, bugfix, hotfix等)，使用斜杠(/)代替;
5. **避免在分支名称的末尾使用连字符**：这是没有意义的，因为连字符分隔单词，最后没有单词要分隔;
6. **最重要的**：使用描述性的、简洁的、清晰的名称来定义分支上做了什么;

**不好的分支命名：**

- `fixSidebar`
- `feature-new-sidebar-`
- `FeatureNewSidebar`
- `feat_add_sidebar`

**好的分支名：**

- feature/new-sidebar
- add-new-sidebar
- hotfix/interval-query-param-on-get-historical-data

## 分支名称约定前缀

有时分支的目的并不明确。它可以是一个新特性、错误修复、文档更新或其他任何东西。为了解决这个问题，通常的做法是在分支名称上使用前缀来快速解释分支的目的。

- **feature**：它表达了一个将被开发的新功能。例如：`feature/add-filters`;
- **release**：用于准备新版本的发布。`release/` 前缀通常用于在合并分支主版本的新更新以创建版本之前执行诸如最后修改和修订之类的任务。例如，`release/v3.3.1-beta`;
- **bugfix**：它表达的信息是，你正在解决代码中的一个bug，而且它通常与一个问题有关。例如，`bugfix/sign-in-flow`;
- **hotfix**：类似于 bugfix，但它与修复生产环境中存在的关键错误有关。例如，`hotfix/cors-error`;
- **docs**：写一些文档。例如，`docs/quick-start`;

如果你在工作中使用任务管理相关的工具，如Jira, Trello, ClickUp，或任何类似的工具，可以考虑先创建用户故事卡，每张卡有一个数字相关联。所以，通常可以把这些卡的编号用在分支名称的前缀中。例如：

- `feature/T-531-add-sidebar`
- `docs/T-789-update-readme`
- `hotfix/T-142-security-path`

## 提交信息 Commit message

接下来，我们来讨论一下提交消息。不幸的是，我们经常会看到这样的提交信息，如：“added a lot of things”或“Pikachu, I choose you”等（是的，我曾经发现一个项目的提交信息与poksammon战斗有关）。

提交信息在开发过程中非常重要，创造一段美好的历史会在你的人生旅途中给你带来很多帮助。和分支一样，社区也有对于提交信息的规范约定，你可以在下面了解到:

- 提交消息有三个重要部分:**主题 Subject**、**描述 Description**和**页脚 Footer**。提交的主题是必需的，并且定义了提交的目的。描述(主体)用于为提交的目的提供额外的上下文和解释。最后是页脚，通常用于元数据，如分配提交。虽然同时使用描述和页脚被认为是一种很好的做法，但这不是必需的。
- **在主题行中使用祈使句**。 例如：

> Add README.md ✅;
> Added README.md ❌;
> Adding README.md ❌;

- **主题行的第一个字母大写**。 例如：

> Add user authentication ✅;
> add user authentication ❌;

- **不要以句号结束主题行**。例如

> Update unit tests ✅;
> Update unit tests. ❌;

- 将主题行限制在**50个字符**以内，也就是说，要清晰简洁;
- 正文以**72个字符**进行换行，并**将主题与空白行分开**;
- 如果你的正文有多个段落，那么**用空行分隔它们**;
- 如有必要，使用**要点**而不是段落;

## 提交规范

> "The Conventional Commits specification is a lightweight convention on top of commit messages. It provides an easy set of rules for creating an explicit commit history."

以下引用来自 Conventional Commit 的官方网站。该规范是社区中最常用的提交消息的约定。

### 结构

```text
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### 提交类型 Commit type

我们要学习的第一个结构是提交类型。它提供了一个清晰的上下文，说明在这个提交中做了什么。下面你可以看到提交类型的列表以及何时使用它们:

- **feat**：引入新功能;
- **fix**：修正软件错误;
- **refactor**：用于代码重构，但保留其整体功能;
- **chore**：不影响生产代码的更新，包括工具、配置或库调整;
- **docs**：对文档文件的补充或修改;
- **perf**：提高性能的代码更改;
- **style**：与代码表现形式相关的调整，如格式和空白;
- **test**：包含或修正测试;
- **build**：影响构建系统或外部依赖的修改;
- **ci**：更改 CI 配置文件和脚本;
- **env**：调整或添加 CI 过程中的配置文件，例如容器配置参数。

### 作用域 Scope

作用域是一个可选的结构，添加到提交类型之后，以提供额外的上下文信息，例如:

- `fix(ui): resolve issue with button alignment`
- `feat(auth): implement user authentication`

### 主体 Body

主体提供了有关提交所引入的更改的详细解释，它通常添加在主题行后面的空白行之后。

示例：

```text
Add new functionality to handle user authentication.

This commit introduces a new module to manage user authentication. It includes
functions for user login, registration, and password recovery.
```

### 页脚 Footer

页脚用于提供与提交相关的附加信息。这可以包括诸如谁审查或批准变更之类的细节。

示例：

```text
Signed-off-by: John <john.doe@example.com>
Reviewed-by: Anthony <anthony@example.com>
```

### 破坏性变更 Breaking Change

表明提交包含可能导致兼容性问题或需要修改相关代码的重大更改。您可以在页脚添加 `BREAKING CHANGE` 或在类型/作用域之后包含 `!` 。

### 使用提交规范的提交示例

```text
chore: add commitlint and husky
chore(eslint): enforce the use of double quotes in JSX
refactor: type refactoring
feat: add axios and data handling
feat(page/home): create next routing
chore!: drop support for Node 18
```

### 包含主题、正文和页脚的示例

```text
feat: add function to convert colors in hexadecimal to rgba

Lorem Ipsum is simply dummy text of the printing and typesetting industry.

Lorem Ipsum has been the industry's standard dummy text ever since the 1500s.

Reviewed-by: 2
Refs: #345
```

## 参考文章

- <https://www.conventionalcommits.org>
- <https://medium.com/@abhay.pixolo/naming-conventions-for-git-branches-a-cheatsheet-8549feca2534>
- <https://se-education.org/guides/conventions/git.html>
- <https://cbea.ms/git-commit/> <https://blog.geekhunter.com.br/o-que-e-commit-e-como-usar-commits-semanticos/>

> 翻译自：<https://dev.to/anthonyvii/be-a-better-developer-with-these-git-good-practices-2dim>
