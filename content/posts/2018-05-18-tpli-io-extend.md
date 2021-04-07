---
title: 'TLPI笔记—深入文件I/O模型'
date: 2018-05-18 9:20:48
categories: 'Linux'
tags: ['tlpi', 'Linux', 'note', 'I/O']
---

## 原子操作和竞争操作

所有的系统调用都是以原子操作方式执行的。之所以这么说，是指内核保证了某系统调用中的所有步骤会作为地理操作而一次性加以执行，期间不会被其他进程或线程中断。原子性规避了**竞争状态（race condition）**，竞争状态指：操作共享资源的两个进程（或线程）其结果取决于一个无法预期的顺序，即这些进程或线程获得 CPU 使用权的先后相对顺序。

<!--more-->

## 文件 I/O 中的竞争状态及规避

### 以独占方式创建一个文件

当同时指定 O_EXCL 与 O_CREATE 作为 `open()`的标志位时，如果要打开的文件已经存在，则返回一个错误。这提供了一个机制，保证进程是打开文件的创建者。对文件是否存在的检查和创建文件属于同一原子操作。要理解这一点的重要性，举个代码示例，该代码中并未使用 O_EXCL 标志：

```c
fd = open(argv[1], O_WRONLY);
if (fd != -1){
    printf("[PID %ld] File \"%s\" already exsits\n",
          (long)getpid(), argv[1]);
}else{
    if (errno != ENOENT){
        errExit("open")
    }else{
        fd = open(argv[1], O_WRONLY|O_CREATE, S_IRUSR|S_IWUSR);
        if (fd == -1)
            errExit("open")

        printf("[PID %ld] Created file \"%s\" exclusively\n",
              (long)getpid(), argv[1]);
    }
}
```

这段代码中除了调用了两次`open()`外，还潜伏着一个 bug。假设如下情况：当第一次调用`open()`时，希望打开的文件还不存在，而当第二次调用`open()`时，其他进程已经创建了该文件。如下图中所示：

![](http://ww1.sinaimg.cn/large/8d56d744ly1frfdkvbdauj20sa0vkn45.jpg)

若内核调度器判断出分配给 A 进程的时间片已经耗尽，并将 CPU 的使用权交给 B 进程，就会发生这种问题。在这一场景下，进程 A 将得出错误的结论:目标文件是有自己创建的。因为无论目标文件存在与否，进程 A 对`open()`的第二次调用都会成功。

由于第一个进程在检查文件是否存在和创建文件之间发送了中断，造成两个进程都声称自己是文件的创建者的问题。结合 O_CEARTE 和 O_EXCL 标志来一次性调用`open()`可以防止这种情况，因此确保检查文件和创建文件的步骤属于单一原子操作。

### 向文件尾部追加数据

多进程同时向同一个文件（例如：全局日志文件）尾部添加数据。为了达到这个目的，也许可以考虑在每个谢金成中使用如下代码：

```c
if (lseek(fd, 0, SEEK_END) == -1)
    errExit("lseek");
if (write(fd, buf, len) != len)
    fetal("Partial/failed write");
```

但是，这段代码存在的缺陷与前一个例子如出一辙。如果第一个进程执行到`lseek()` 和 `write()` 之间，被执行相同代码的第二个进程所中断，那么这两个进程会在写入数据前，将文件偏移量设为相同的位置，而当第一个进程再次获得调度时，会覆盖第二个进程已写入的数据。此时，再次出现竞争状态。

要规避这个问题，需要将文件偏移量的移动于数据写入操作纳为同一原子操作。在打开文件时，加入 O_APPENDB 标志就可以保证这一点。

## 文件描述符合打开文件之间的关系

文件描述符和已经打开的文件之间并不一定都是一一对应的关系。多个文件描述符指向同一打开文件，这既有可能，也属必要。这些文件描述符可在相同或不同的进程中打开。

要理解具体情况，需知道由内核维护的三个数据结构：

1、进程级的**文件描述符表** (file description)

(1)控制文件描述符操作的一组标志（目前此标志仅定义一个 close-on-exec 标志）

(2)对打开文件句柄的引用

2、系统级的**打开文件表** (open file table)
(1)当前文件偏移量(off_set)

(2)打开文件时所使用的状态标志

(3)文件访问模式

(4)与信号驱动 I/O 相关的设置

(5)对该文件 i-node 对象的引用

3、文件系统的**i-node 表**
(1)文件类型（例如，常规文件，套接字和 FIFO 等）和访问权限

(2)一个指针，指向该文件所持有的锁的列表

(3)文件的各种属性，包括文件大小，不同类型操作的时间戳等、

三者之间的关系如下图所示：

![](http://ww1.sinaimg.cn/large/8d56d744ly1frfeqceqnuj216q0ti47m.jpg)

在进程 A 中，文件描述符 1 和 20 都指向同一个打开的文件句柄（标号为 23）。这可能是通过调用`dup()`,`dup2()`,或`fcntl()`而形成的。

进程 A 的文件描述符 2 和进程 B 的文件描述符 2 都指向同一个打开的文件句柄 73，这种情况可能是在调用`fork()`后出现，或者当某进程通过 UNIX 域套接字将一个打开的文件描述符传递给另一个进程时，也会发生。

进程 A 的描述符 0 和进程 B 的描述符 3 分别指向不同的打开文件句柄，但这些句柄指向相同的 i-node 条目 1976，也就是指向同一个文件。发生这种情况是因为每个进程各自对同一个文件发起了`open()`调用。

## 文件 I/O 操作的常用系统调用

### 文件控制操作 fcntl()

对一个已经打开的文件描述符执行一系列控制操作

```c
#include <fcntl.h>

int fcntl(int fd, int cmd, ...);
# Return on success depends on cmd, or -1 on error
```

要想获取一个打开文件描述符的 flag 参数设置，就将`fcntl()`的 cmd 参数设置为**F_GETFL**。

```c
int flags, accessMode;

flags = fcntl(fd, F_GETFL);
if (flags == -1)
    errExit("fcntl");

# 获取到flags以后，可以通过与的方式修改，比如要以同步写方式打开
if (flags & O_SYNC)
    printf("success\n")
```

### 复制文件描述符 dup() 和 dup2()

`dup()`调用复制一个打开的文件描述符 oldfd，并返回一个新描述符，二者都指向同一打开的文件句柄。系统会保证新描述符一定是编号值最低的未用文件描述符。

```c
#include <unistd.h>

int dup(int oldfd);
# Return (new) fd on success, or -1 on error
```

`dup()`是系统返回一个文件描述符，如果想控制获得的新的文件描述符，可以调用`dup2()`。

`dup2()`系统调用会为 oldfd 参数所指定的文件描述符创建副本，副本的文件件描述符由 newfd 参数指定。

```c
#include <unistd.h>

int dup2(int oldfd, int newfd);
# Return (new) fd on success, or -1 on error
```

### 在文件特定位置处的 I/O pread() 和 pwrite()

`pread()`和`pwrite()`所完成的工作和`read()`,`write()`相类似，只是前两者会在 offset 参数所指定的位置进行文件 I/0 操作，而非始于文件的当前偏移量处，且**它们不会改变文件的当前偏移量**。

```c
#include <unistd.h>

ssize_t pread(int fd, void *buf, size_t count, off_t offset);
# Return number of bytes read, 0 on EOF, or -1 on error

ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);
# Return number of bytes written, or -1 on error
```

`pread()`调用等同于将如下操作纳入同一原子操作：

```c
off_t orig;

orig = lseek(fd, 0, SEEK_CUR); # 保存当前文件偏移量
lseek(fd, offset, SEEK_SET); # 挪动文件偏移量
s = read(fd, buf, len); # 读取指定长度的数据
lseek(fd, orig, SEEK_SET); # 将文件偏移量恢复到初始操作点
```

### 分散输入和集中输出 readv()和 writev()

`readv()`和`writev()`系统调用分别实现了分散输入和集中输出的功能。（就是分块读取和多块写入）

```c
#include <sys/uio.h>

ssize_t readv(int fd, const struct iovec *iov, int iovcnt);
# Return number of bytes read, 0 on EOF, -1 on error

ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
# Return number of written, or -1 on error
```

这些系统调用并非一次只对单个缓冲区进行读写操作，而是一次可以传输多个缓冲区的数据进去。数组 iov 定义了一组用来传输数据的缓冲区。iovcnt 则指定了 iov 的成员个数。

```c
# iovec结构体的定义
struct iovec{
    void *iov_base; # 数据缓冲区的起始地址指针
    size_t iov_len; # 数据缓冲区的长度
}
```

关于 iov，iovcnt 以及缓冲区之间的关系如图：

![](http://ww1.sinaimg.cn/large/8d56d744ly1frfh8zd3hhj211c0cm0u4.jpg)

### 截断文件 truncate() 和 ftruncate()

`truncate()`和`ftruncate()`系统调用将文件大小设置为 length 长度指定的值。

```c
#include <unistd.h>

int truncate(const char* pathname, off_t length);
int ftruncate(int fd, off_t length);

# Return 0 on sunccess, or -1 on error
```

若文件当前长度大于参数 length，调用将丢弃超出的部分，若小于，调用将在文本尾部添加一系列空字节或是一个文件空洞。

> `truncate()`无需先以`open()`来获取文件描述符，却可以修改文件内容，在系统调用中可谓独树一帜

### 创建临时文件 mktemp() 和 tmpfile()

`mktemp()`函数生成为一个额文件名并打开文件，返回一个可用于 I/O 调用的文件描述符。

```c
#include <stdlib.h>

int mkstemp(char *template);
# Return fd on sunccess, or -1 on error
```

template 是模板参数，采用路径名形式，其中最后刘哥字符必须为**XXXXXX**, 这 6 个字符将被替换，以保证文件名的唯一性，且修改后的字符串将通过 template 参数传回。因为会修改 template，所以必须将其指定为字符数组，而非字符串常量。

`tmpfile()`函数会创建一个名称为一的临时文件，并以读写方式打开。但该函数执行成功，将返回一个文件流，提供 stdio 库函数使用，内部会自动调用`unlink()`系统调用，文件流关闭后悔自动删除临时文件。

```c
#include <stdio.h>

FILE *tmpfile(void);
# Return file pointer on success, or NULL in error
```

## 大文件 I/O

始于内核版本 2.4，32 位 Linux 系统开始提供对 LFS(大型文件峰会—Large File Summit)的支持（glibc 2.2+）。另一个前提是相应的文件系统也必须支持大文件操作。

### 过渡性 API

刚开始支持 LFS 的时候，是通过使用一些过渡性 API 来实现，这些 API 所属函数具有处理 64 位文件大小和文件偏移量的能力。这些函数与 32 位版本的命名相同，只是尾部缀以 64 以示区别。比如：**fopen64()、open64()、lseek64()**等等。

### \_FILE_OFFSET_BITS 宏

要获取 LFS 功能，推荐的做法是：在编译程序时，将宏**\_FILE_OFFSET_BITS**的值定义为 64。具体实现方式有两种：

- 利用 C 语言编译器的命令行 `$ cc -D_FILE_OFFSET_BITS=64 prog.c`
- 在 C 语言源文件中，包含所有头文件之前添加宏 `#define __FILE_OFFSET_BITS 64`

## /dev/fd 目录

对于每个进程，内核都提供一个特殊的虚拟目录**/dev/fd**。该目录中包含”/dev/fd/n“形式的文件名，其中 n 是与进程中的打开文件描述符相对应的编号。例如 /dev/fd/0 就对应进程的标准输入。

> /dev/fd 实际上是一个符号链接，链接到 Linux 所专有的/proc/self/fd 目录。后者又是 Linux 特有的/proc/PID/fd 目录族的特例之一，此目录族中的而每一个目录都包含有符号链接，与一进程所打开的所有文件相对应。

——EOF——
