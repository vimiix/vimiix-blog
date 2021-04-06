---
title: 'pipenv错误解决:TypeError: "module" object is not callable'
date: 2018-10-21 11:22:48
categories: 'Python'
tags: ['Python', 'note', 'pipenv', 'solution']
---

### 软件版本

今天在折腾一台新的云主机，所以我在安装环境的时候`pip`和`pipenv`都选择安装了最新版本

（_注：正是这两个版本配合才会出现下面的报错，旧版本或以后的新版本的 Pipenv 不会出现_）。

具体如下：

```bash
// pipenv 的版本 2018.7.1
$ pipenv --version
pipenv, version 2018.7.1

// pip 的版本 18.1
$ pip --version
pip 18.1 from /usr/bin/python3.6/lib/python3.6/site-packages/pip (python 3.6)
```

<!--more-->

### 报错日志

```bash
$ pipenv install
Installing dependencies from Pipfile.lock (2a98a1)...
Traceback (most recent call last):▉▉▉▉ 0/19 — 00:00:00
  File "/usr/bin/python3.6/bin/pipenv", line 11, in <module>
    sys.exit(cli())
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/vendor/click/core.py", line 722, in __call__
    return self.main(*args, **kwargs)
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/vendor/click/core.py", line 697, in main
    rv = self.invoke(ctx)
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/vendor/click/core.py", line 1066, in invoke
    return _process_result(sub_ctx.command.invoke(sub_ctx))
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/vendor/click/core.py", line 895, in invoke
    return ctx.invoke(self.callback, **ctx.params)
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/vendor/click/core.py", line 535, in invoke
    return callback(*args, **kwargs)
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/cli.py", line 435, in install
    selective_upgrade=selective_upgrade,
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/core.py", line 1943, in do_install
    pypi_mirror=pypi_mirror,
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/core.py", line 1322, in do_init
    pypi_mirror=pypi_mirror,
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/core.py", line 807, in do_install_dependencies
    pypi_mirror=pypi_mirror,
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/core.py", line 1375, in pip_install
    package_name.split('--hash')[0].split('--trusted-host')[0]
  File "/usr/bin/python3.6/lib/python3.6/site-packages/pipenv/vendor/requirementslib/models/requirements.py", line 704, in from_line
    line, extras = _strip_extras(line)
TypeError: 'module' object is not callable
  🐍   ▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉▉ 0/19 —
```

### 解决方案 1 - 降 pip 版本

这个错误是由于`pipenv`和最新版本的**pip（18.1）**一起配合使用导致的 pipenv 中的一个错误，所以解决方法就是将 pip 的版本降到**18.0**即可。（问题讨论见[issue 2924](https://github.com/pypa/pipenv/issues/2924)）

执行下面的命令：

```bash
sudo pip install pip==18.0
```

### 解决方案 2 - 升 pipenv 版本

上面的做法是自行解决的方案，还有一种方案就是更新到最新版的**pipenv 2018.10.9** （新版本中该问题被得到解决 [#2935](https://github.com/pypa/pipenv/pull/2935)）

具体的解决方案，参考官方的[comment](https://github.com/pypa/pipenv/issues/2924#issuecomment-429573525)：

```markdown
Also you can upgrade pipenv to the latest version.
If you have further problems please open a new issue.
I’m going to lock this thread for now to prevent
continued discussion as this is resolved.

To summarize for people on earlier releases than `2018.10.9`:
you can pin pip inside the virtualenv using
`pipenv run python -m pip install pip==18.0`
or you can set the environment variable
`PIP_SHIMS_BASE_MODULE=pipenv.patched.notpip`
```

不过可能是我使用的 pip 源是清华的镜像站，截止到发文，该镜像站还未同步到官方指出的`2018.10.9`版本，懒得折腾了，所以采用了**降 pip 版本**的方案。

--- EOF ---
