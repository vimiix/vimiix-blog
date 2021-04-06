---
title: 'beanstalkd消息队列'
date: 2020-03-11 16:22:48
categories: 'beanstalkd'
tags: ['beanstalkd', 'note', 'Python', 'MQ']
---

beanstalkd 是一个简单快速的分布式工作队列系统，协议基于 ASCII 编码运行在 tcp 上。其最初设计目的是通过后台异步执行耗时任务方式降低高容量 Web 应用的页面延时。而其简单、轻量、易用等特点，和对任务优先级、延时/超时重发等控制，以及[众多语言版本的客户端](https://github.com/beanstalkd/beanstalkd/wiki/Client-Libraries)的良好支持，使其可以很好的在各种需要队列系统的场景中应用。

Beanstalk 的应用场景主要有：

- 消息异步处理（消息队列的基本需求）
- 消息延迟处理，实现循环队列

### beanstalkd 核心组件

- `job` : 任务，队列中的基本单元
- `tube` : 一个有名称的任务队列，用来存储统一类型的 `job` ,beanstalkd 通过 `tube` 来实现多任务队列
- `producer` : `job` 生产者，通过`put`命令来创建一个`job`放到一个`tube`中
- `comsumer` : `job` 消费者，通过 `reserve`、`release`、`bury`、`delete`命令来获取或改变 `job` 的状态

### job 的生命周期

在整个生命周期中 job 可能有四种工作状态：`READY`、`RESERVED`、`DELAYED`、`BURIED`. 只有处于 `READY` 状态的 job 才能被消费。

```
 put with delay            release with delay
  -------------> [DELAYED] <---------.
                     |                   |
                     | (time passes)     |
                     |                   |
   put               v     reserve       |       delete
  --------------> [READY] -------> [RESERVED] ------> *poof*
                     ^  ^                |  |
                     |   \  release      |  |
                     |    `-------------'   |
                     |                      |
                     | kick                 |
                     |                      |
                     |       bury           |
                  [BURIED] <---------------'
                     |
                     |  delete
                      `--------> *poof*
```

Producer 创建 job 的时候可以选择两种方式：`put` , `put with delay`。

当 producer 直接 `put` 一个 job 时，job 就处于 `READY` 状态，等待 consumer 来处理，如果选择 `put with delay`，job 的初始状态为 `DELAYED` 状态，等待延迟时间过后才迁移到 READY 状态。两种创建 job 的方式都会传入一个 TTR（Timeout 机制），当 job 处于 `RESERVED` 状态时，TTR 开始倒计时。如果 TTR 倒计时完，job 状态还没有变，就可以认为 job 处理失败，会被重新放回队列。

consumer 获取（reserve）了一个 READY 的 job 后，该 job 的状态就迁移到 `RESERVED`。这时候，其他的 consumer 就不能再操作该 job 了。当 consumer 完成该 job 后，可以选择 delete , release 或者 bury 操作;

- delete 操作，job 从系统消亡，之后不能再获取;

- release 操作，可以重新把该 job 状态迁移回 `READY` ，使其他的 consumer 可以继续获取和执行该 job，也可以延迟（release with delay）该状态迁移操作，这样会中间会多一步 `DELAYED` 状态的转换。

- bury 操作，可以把该 job 休眠，等到需要的时候，再将休眠的 job 通过 kick 命令迁移回 `READY` 状态，也可以通过 delete 销毁 `BURIED` 状态的 job。

正是有这些有趣的操作和状态，才可以开篇提到的多种应用场景，比如要实现类似 crontab 功能的循环队列，就可以将 `RESERVED` 状态的 job 休眠掉，等没有 `READY` 状态的 job 时再将 `BURIED` 状态的 job 一次性 kick 回 `READY` 状态。

为什么要有 `BURIED` 状态？一个是可以实现 job 不依赖延迟的重复使用，以实现循环队列的功能，另一方面，这其实也可以作为一种异常捕获机制，比如对于前面处理超时的 job，我们就可以将其设置为 `BURY` 状态，这样有两个好处：

1. 避免重新放回队列，导致主流程再次进入到异常
2. 方便后期人工排查错误

### beanstalkd 的一些特性

- 优先级，producer 产生的任务可以给他分配一个优先级，支持 0 到 2^32 的优先级，值越小，优先级越高，默认优先级为 1024。优先级高的会被消费者首先执行
- 持久化，可以通过 binlog 将 job 及其状态记录到文件里面，在 beanstalkd 下次启动时可以通过读取 binlog 来恢复之前的 job 及状态
- 分布式，分布式设计和 Memcached 类似，beanstalkd 各个 server 之间并不知道彼此的存在，都是通过 client 来实现分布式以及根据 tube 名称去特定 server 获取 job
- 超时机制，为了防止某个 consumer 长时间占用任务但不能处理的情况，beanstalkd 为 reserve 操作设置了 timeout 时间，如果该 consumer 不能在指定时间内完成 job，job 将被迁移回 `READY` 状态，供其他 consumer 执行。

### 一些链接

- beanstalkd 源码地址： [https://github.com/beanstalkd/beanstalkd](https://github.com/beanstalkd/beanstalkd)
- 各编程语言客户端列表：https://github.com/beanstalkd/beanstalkd/wiki/Client-Libraries
- beanstalkd 中文协议：https://github.com/beanstalkd/beanstalkd/blob/master/doc/protocol.zh-CN.md
