---
title: 'Shell 函数实现Go语言多版本管理轻量级方案'
date: 2023-06-21 14:50:48
categories: 'golang'
tags:
  - 'golang'
  - 'solution'
  - 'multi-version'
---

## 现有的工具方案

- gvm: <https://github.com/moovweb/gvm>
- g: <https://github.com/voidint/g>

## 我的方案

优点：

- 原生：基于 go 语言本身支持多版本的能力实现，可以下载任何官方发布的版本
- 简单：shell 函数实现，直接集成到 bashrc 或 zshrc 中即可使用，无需额外配置
- 可定制化：代码简单可根据自身需求定制

## 代码实现

gist地址：[https://gist.github.com/vimiix/0927fdfbea926e869a2c631db9eeae8b](https://gist.github.com/vimiix/0927fdfbea926e869a2c631db9eeae8b)

```shell
####### GOLANG VERSION MANAGE FUNCTIONS ######
# ref: https://go.dev/doc/manage-install
function goinstall() {
 echo "Downloading go$1 ..."
 go install golang.org/dl/go$1@latest && go$1 download
}

function gouse() {
 gopath=$(go env GOPATH)
 if test -x ${gopath}/bin/go$1; then
  rm -f ${gopath}/bin/go
  echo "Relink go with go$1 ..."
  ln -s ${gopath}/bin/go$1 ${gopath}/bin/go
  echo "Done"
 else
  echo "Version $1 not installed"
 fi
}

function golist() {
 current=$(go version | awk '{print $3}' | cut -c3-)
 for v in $(ls $(go env GOPATH)/bin | grep -E 'go(\d.*)' | cut -c3-);
 do
  if [ $v = $current ]; then
   echo "$v (⇦ current)"
  else
   echo $v
  fi
 done
}

function gouninstall() {
 current=$(go version | awk '{print $3}' | cut -c3-)
 if [ $1 = $current ]; then
  echo "version $1 is actived, please change to another version first"
  return
 fi
 echo "Removing binary..."
 rm -f $(go env GOPATH)/bin/go$1
 echo "Removing sdk ..."
 rm -r ~/sdk/go$1
 echo "Done"
}
```

## 使用方式

1. 将上面的代码粘贴到 `~/.bashrc` （如果是zsh，则是 `~/.zshrc`） 末尾，保存退出
2. `source ~/.bashrc` 激活即可

## 使用演示

[![asciicast](https://asciinema.org/a/592495.svg)](https://asciinema.org/a/592495)
