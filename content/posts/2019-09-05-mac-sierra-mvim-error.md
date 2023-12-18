---
title: 'Mac 升级10.12.6 mvim 打开文件报错'
date: 2019-09-05 16:22:48
categories: 'solution'
tags: ['solution', 'mac', 'mvim']
---

> 当昨天把 Mac 升级了 10.12.6 Sierra 以后，mvim 打开文件的时候就开始报错，使用该方法已解决~

### 报错信息

```bash
dyld: Library not loaded: /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/libruby.2.0.0.dylib
  Referenced from: /usr/local/Cellar/macvim/8.0-146/MacVim.app/Contents/bin/../MacOS/Vim
  Reason: image not found
[1]    33114 abort      mvim -v
```

<!--more-->

### 解决方法

这个错误是 [macvim](https://github.com/macvim-dev/macvim) 报的错，并非 [vim](https://www.vim.org/).

```bash
➜ vimiix  ~  type vim
vim is an alias for mvim -v
```

我这里使用的 vim 指令是对 `mvim` 的一个别名。

使用一条指令可以解决上面的报错问题：

```bash
sudo install_name_tool -change /System/Library/Frameworks/Ruby.framework/Versions/2.0/usr/lib/libruby.2.0.0.dylib /System/Library/Frameworks/Ruby.framework/Versions/2.3/usr/lib/libruby.2.3.0.dylib /usr/local/Cellar/macvim/8.0-146/MacVim.app/Contents/bin/../MacOS/Vim
```

#### 注意点

需要注意的是，指令最后指定的 `vim` 路径，一定是上面报错中 `Referenced from` 后面的路径。[参考](https://stackoverflow.com/questions/47278449/vim-ruby-mismatch-on-mac-high-sierra#comment93136732_49450569)

### 相关解答

- [https://stackoverflow.com/questions/47278449/vim-ruby-mismatch-on-mac-high-sierra](https://stackoverflow.com/questions/47278449/vim-ruby-mismatch-on-mac-high-sierra)
