---
title: '磁盘只读（readonly）故障场景模拟'
date: 2021-09-01 16:15:48
categories: 'linux'
tags:
  - 'disk'
  - 'simulate'
  - 'test'
---

假设服务器目前有多个盘，`vdb1`这块分区盘专门用于数据库程序的数据目录，我们就用 `vdb1` 这个盘来模拟只读故障场景。

![image-20210901154323971](https://static.vimiix.com/upic/2021-09-01/image-20210901154323971.png)

### 1. 卸载指定盘

```bash
umount /dev/vdb1
```

想要如期卸载掉，需要确保该盘上没有被正在运行进程依赖，如果有运行中的进程依赖这个盘，会报如下报 `target is busy` 的错误：

```bash
umount: /opt: target is busy.
```

遇到该错误时，可以通过`lsof [mountpoint]`  命令来查看有哪些进程依赖这块盘，kill 掉相应的进程后重新卸载。

<!--more-->

### 2. 配置伪设备来镜像 /dev/vdb1 的内容

卸载成功后，我们通过 `losetup` 指令来配置一个loop设备（*在类 UNIX 系统里，loop 设备是一种伪设备(pseudo-device)，或者也可以说是仿真设备。它能使我们像块设备一样访问一个文件*）。

```bash
losetup /dev/loop0 /dev/vdb1
```

执行指令后，可以通过 `losetup -l`  查看当前服务器上的 loop 设备列表：

![image-20210901155336595](https://static.vimiix.com/upic/2021-09-01/image-20210901155336595.png)

### 3. 挂载 /dev/loop0

我们已经有一个 `/dev/loop0` 的伪设备镜像到了 `/dev/vdb1` 上，此时，我们来挂载 `/dev/loop0` 这个伪设备：

```bash
mount /dev/loop0 /opt -o rw,errors=remount-ro
```

通过 `df -h`  可以查看到 `/dev/loop0` 已经正常挂载上了

![image-20210901160108398](https://static.vimiix.com/upic/2021-09-01/image-20210901160108398.png)

### 4. 正常启动应用进程（这里测试用opengauss数据库）

```bash
gs_ctl start -D /opt/mogdb/data/
```

### 5. 修改磁盘为只读模式（故障注入）

`blockdev` 命令行工具是用于控制区块设备程序

```bash
blockdev --setro /dev/vdb1
```

此时，尝试写入 `/dev/loop0` 设备，将导致I/O错误。一旦文件系统驱动程序检测到，它将以只读方式重新挂载文件系统。

这样就实现了磁盘只读故障的场景模拟。

### 6. 测试后恢复系统

a. 取消挂载

```bash
umount /opt
```

b. 删除伪设备

```bash
losetup -d /dev/loop0
```

c. 将 `/dev/vdb1` 设备恢复为可读写

```bash
blockdev --setrw /dev/vdb1
```

d. 将 `/dev/vdb1` 重新挂载回来

```bash
mount /dev/vdb1 /opt
```

--- EOF ---
