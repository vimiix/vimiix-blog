---
title: 'Golang实现Windows系统用户和密码校验'
date: 2021-03-22 16:22:48
categories: 'golang'
tags: ['note', 'windows', 'golang']
---

本质上是通过调用 windows 的一个 API —— [`LogonUserW`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-logonuserw) ，来实现对于用户密码的校验。

> 仅适用于在本地校验，不支持远程连接校验

用一个示例代码来进行说明，下面是目录结构中，`main.go` 是程序入口文件，auth 包中，我们仅实现 windows 系统的校验代码，其他平台不属于本文介绍内容，就直接返回 nil 即可。

<!--more-->

#### 目录结构

```bash
.
├── auth
│   ├── auth.go
│   └── auth_windows.go
├── go.mod
└── main.go
```

#### 源代码 [[src]](https://github.com/vimiix/authDemo)

- main.go

```go
package main

import (
    "flag"
    "fmt"
    "os"

    "github.com/vimiix/authDemo/auth"
)

func main() {
    user := flag.String("u", "", "username")
    password := flag.String("p", "", "password")
    flag.Parse()

    if *user == "" || *password == "" {
        fmt.Println("Both user and password should be specify")
        fmt.Printf("Usage: %s -u [username] -p [password]\n", os.Args[0])
        return
    }


    err := auth.Auth(*user, *password)
    if err != nil {
        fmt.Printf("Auth failed.\nError message:%v\n", err)
    } else {
        fmt.Println("Auth successfully")
    }
}

```

- auth_windows.go

```go
// +build windows

package auth

import (
    "syscall"
    "unsafe"
)

// 下面的常量定义在 Winbase.h 头文件中
// logon type
const (
    LOGON32_LOGON_INTERACTIVE = 2
)

// logon provider
const (
    LOGON32_PROVIDER_DEFAULT = 0
)

var (
    modAdvApi32    = syscall.NewLazyDLL("advapi32.dll")
    procLogonUserW = modAdvApi32.NewProc("LogonUserW")
)

func Auth(user, password string) error {
    return nil
}

func logon(username string, domain string, password string) (syscall.Token, error) {
    var token syscall.Token
    err := logonUserW(
        syscall.StringToUTF16Ptr(username),
        syscall.StringToUTF16Ptr(domain),
        syscall.StringToUTF16Ptr(password),
        LOGON32_LOGON_INTERACTIVE,
        LOGON32_PROVIDER_DEFAULT,
        &token,
    )
    if err != nil {
        return 0, err
    }

    return token, nil
}

func logonUserW(username *uint16, domain *uint16, password *uint16, logonType uint32,
    logonProvider uint32, outToken *syscall.Token) error {
 ret, _, err := syscall.Syscall6(
  procLogonUserW.Addr(),
  6,
  uintptr(unsafe.Pointer(username)),
  uintptr(unsafe.Pointer(domain)),
  uintptr(unsafe.Pointer(password)),
  uintptr(logonType),
  uintptr(logonProvider),
  uintptr(unsafe.Pointer(outToken)),
 )
 if ret == 0 {
  if err == 0 {
   err = syscall.EINVAL
  }
  return err
 }
 return nil
}

```

- auth.go

```go
// +build !windows

package auth

func Auth(user, password string) error {
    return nil
}
```

#### 编译

```bash
GOOS=windows GOARCH=amd64 go build -o auth.exe main.go
```

#### Usage

![image-20210322221809864](https://static.vimiix.com/uPic/2021-03-22/image-20210322221809864.png)

#### golang 相关的库

- <https://github.com/golang/sys/blob/77190671981848ff71219ecfff38636c9add3b79/windows/dll_windows.go>
- <https://github.com/itchio/ox>
- <https://github.com/lxn/win>
- <https://github.com/JamesHovious/w32>

#### 参考链接

- <https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-logonuserw>

- <https://knowledgebase.progress.com/articles/Article/P21685>
- <https://bbs.csdn.net/topics/360202554>
