---
title: 'HLS视频加密'
date: 2019-03-28 16:22:48
categories: 'note'
tags: ['note', 'video', 'encrypt', 'ffmpeg', 'hls']
---

> 最近在做视频管理后台，主要提供点播服务，涉及到需要对视频进行加密处理以防止视频被随意下载。
>
> 调研了一番之后确定使用 HLS(HTTP Live Streaming) 基于 HTTP 的流媒体网络传输协议技术来处理视频。
>
> 所以本文主要记录关于学习 HLS 视频加密技术的笔记

### 为什么要加密？

简单的说就是：增加获取被加密资源的代价。对于视频这种资源来说，绝对的加密就是不要上线给人看，但那是不可能的，因为提供的服务就是给人看视频，只要上线，别人就可以通过各种手段解密或者简单的录屏的方式来传播，所以目前俩看，不存在绝对的加密。只要让恶意的人获取源视频的代价很大，就可以阻挡绝大多数的不法分子。这样，加密的目的也就基本达到了。

<!--more-->

### 什么是 HLS？

**HTTP Live Streaming**（缩写是**HLS**）是一个由苹果公司提出的基于[HTTP](https://zh.wikipedia.org/wiki/HTTP)的[流媒体](https://zh.wikipedia.org/wiki/%E6%B5%81%E5%AA%92%E4%BD%93)[网络传输协议](https://zh.wikipedia.org/wiki/%E7%BD%91%E7%BB%9C%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)。是苹果公司[QuickTime X](https://zh.wikipedia.org/w/index.php?title=QuickTime_X&action=edit&redlink=1)和[iPhone](https://zh.wikipedia.org/wiki/IPhone)软件系统的一部分。它的工作原理是**把整个流分成一个个小的基于 HTTP 的文件来下载，每次只下载一些**。当媒体流正在播放时，客户端可以选择从许多不同的备用源中以不同的速率下载同样的资源，允许流媒体会话适应不同的数据速率。在开始一个流媒体会话时，客户端会下载一个包含元数据的[extended M3U (m3u8)](https://zh.wikipedia.org/w/index.php?title=Extended_M3U&action=edit&redlink=1) [playlist](https://zh.wikipedia.org/w/index.php?title=Playlist&action=edit&redlink=1)文件，用于寻找可用的媒体流。 —— **wikipedia**

![原理图](http://ww1.sinaimg.cn/large/8d56d744ly1g1io6gkhnfj22740v678x.jpg)

HLS 流媒体加密技术的核心就在于将视频切分为一小块一小块的片段（_.ts 文件_），对这每一小块视频片段分别使用[对称加密算法](https://zh.wikipedia.org/wiki/%E5%B0%8D%E7%A8%B1%E5%AF%86%E9%91%B0%E5%8A%A0%E5%AF%86)，在服务端加密，客户端解密，通过权限验证的用户才能拿到解密一小块视频的密钥。

### 为什么使用对称加密？

现代成熟的加密技术分为对称加密算法和公钥密码算法(非对称加密)。之所以选择对称加密是因为流媒体要求很强的实时性，数据量又很大。公钥密码算法的计算都比较复杂，效率较低，适合对少量数据进行加密。对称加密效率相对较高，所以流媒体加密首选对称加密。例如在 SSH 登入的时候会先通过公钥密码算法传输一个密钥，再用这个密钥用作对称加密算法的密钥，在数据传输过程中使用对称加密算法来提示数据传输效率。<sup>[[引用]](https://github.com/gwuhaolin/blog/issues/10)</sup>

### HLS 的优势

- 建立在 HTTP 之上，使用简单，接入代价小

- 分片技术有利于 CDN 加速技术的实施

- 支持点播和录播

- 根据网络带宽变化来智能响应切换流

- 多种不同比特率的流可供选择

### HLS 相关文件解析

HLS 由两部分构成，一个是 .m3u8 索引描述文件，一个是 .ts 媒体文件（TS 是视频文件格式的一种）。

整个过程是，浏览器会首先去请求 .m3u8 的索引文件，然后解析 m3u8 文件，找出对应的 .ts 文件链接，并开始下载。

#### m3u8 索引描述文件

m3u8 是一个文本文件，用文本方式对媒体文件进行描述，由一系列标签组成，核心是一个 .ts 文件的列表，也就是告诉浏览器可以播放这些 ts 文件。

举个例子：

```
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:11
#EXT-X-MEDIA-SEQUENCE:0
#EXT-X-PLAYLIST-TYPE:VOD
#EXT-X-KEY:METHOD=AES-128,URI="https://example.com/key"
#EXTINF:10.416667,
part_0000.ts
#EXTINF:1.458333,
part_0001.ts
#EXTINF:3.875000,
part_0002.ts
#EXTINF:7.291667,
part_0003.ts
#EXT-X-ENDLIST
```

现在针对这个文件做一下解析：

- EXTM3U

  每个 M3U 文件第一行必须是这个 tag，提供标示作用

- EXT-X-VERSION

  用以标示协议版本。这里是 3， 那么这里用的就是 HLS 协议第三个版本，此标签只能有 0 或 1 个，不写代表使用版本 1

- EXT-X-TARGETDURATION

  所有切片的最大时长，有些 Apple 设备这个参数不正确会无法播放。

- EXT-X-MEDIA-SEQUENCE

  切片的开始序号。每一个切片都有唯一的序号，相邻之间序号+1。这个编号会继续增长，保证流的连续性。

- EXT-X-PLAYLIST-TYPE

  类型， vod 表示点播

- EXT-X-KEY:METHOD=AES-128,URI="https://example.com/key"

  这个参数指定了视频的加密算法为 `AES-128`，以及获取密钥的链接地址，播放器将从该位置检索密钥以解密媒体片段，为保护密钥免受窃听，应通过 HTTPS 提供服务，且可能还需要实施一些身份验证机制来限制谁可以访问密钥

- EXTINF: \<duration\>,\<title\>

  duration : 视频时长

- EXT-X-ENDLIST

  文件结束符号。表示不再向播放列表文件添加媒体文件

#### ts 媒体文件

ts 文件是一种传输流文件，视频编码主要格式 h264/mpeg4，音频为 acc/MP3。

ts 文件分为三层：

- es 层 (Elementary Stream 基本码流)，音视频数据
- pes 层 (Packet Elemental Stream 节目流)，在音视频数据上加了时间戳等对数据帧的说明信息
- ts 层 (Transport Stream 传输流)，在 pes 层加入数据流的识别和传输必须的信息

![](http://ww1.sinaimg.cn/large/8d56d744ly1g1je6qlbxmj20i30b80t2.jpg)

##### ts 层

ts 包大小固定为 188 字节，ts 层分为三个部分：

- ts heeader ：固定 4 个字节；
- adaptation field ：可能存在也可能不存在，主要作用是给不足 188 字节的数据做填充；
- payload ： pes 层的数据；

**ts header**

| 标志位                       | bit | 解释                                                                                                                               |
| ---------------------------- | --- | :--------------------------------------------------------------------------------------------------------------------------------- |
| sync_byte                    | 8   | 同步字节，固定为 `0x47`                                                                                                            |
| transport_error_indicator    | 1   | 传输错误指示符，表明在 ts 头的 adapt 域后由一个无用字节，通常都为`0`，这个字节算在 adapt 域长度内                                  |
| payload_unit_start_indicator | 1   | 负载单元起始标示符，一个完整的数据包开始时标记为 `1`                                                                               |
| transport_priority           | 1   | 传输优先级，`0` 为低优先级，`1` 为高优先级，通常取 `0`                                                                             |
| **PID**                      | 13  | **Packet ID 号码，唯一的号码对应不同的包**                                                                                         |
| transport_scrambling_control | 2   | 传输加扰控制，`00` 表示未加密                                                                                                      |
| adaptation_field_control     | 2   | 是否包含自适应区，`00` 保留；`01` 为无自适应域，仅含有效负载；`10` 为仅含自适应域，无有效负载；`11` 为同时带有自适应域和有效负载。 |
| continuity_counter           | 4   | 包递增计数器，从 0-f，起始值不一定取 0，但必须是连续的                                                                             |

加粗的 PID 是 ts 层中唯一识别标志，这个包是什么内容就是由 PID 决定的。下表给出了一些表的 PID 值，这些值是固定的，不允许用于更改。

| 表           | PID 值 |
| ------------ | ------ |
| PAT          | 0x0000 |
| CAT          | 0x0001 |
| TSDT         | 0x0002 |
| EIT, ST      | 0x0012 |
| RST, ST      | 0x0013 |
| TDT, TOT, ST | 0x0014 |

ts 层的内容是通过 PID 值来标识的，主要内容包括：**PAT 表**（Program Association Table，节目关联表）、**PMT 表**（Program Map Table，节目映射表）、**音频流**、**视频流**。解析 ts 流要先找到 PAT 表，只要找到 PAT 就可以找到 PMT，然后就可以找到音视频流了。PAT 表的 PID 值固定为`0x0000`。PAT 表和 PMT 表需要定期插入 ts 流，因为用户随时可能加入 ts 流，这个间隔比较小，通常每隔几个视频帧就要加入 PAT 和 PMT。PAT 和 PMT 表是必须的，还可以加入其它表如 **SDT 表**（Service Descriptor Table，业务描述表）等，不过 hls 流只要有 PAT 和 PMT 就可以播放了。

- PAT 表：他主要的作用就是指明了 PMT 表的 PID 值;
- PMT 表：他主要的作用就是指明了音视频流的 PID 值;
- 音频流/视频流：承载音视频内容;

**adaption**

| 标志位                  | bit | 解释                                                                                                   |
| ----------------------- | --- | ------------------------------------------------------------------------------------------------------ |
| adaptation_field_length | 1   | 自适应域长度，后面的字节数                                                                             |
| flag                    | 1   | 取 `0x50` 表示包含 PCR 或 `0x40` 表示不包含 PCR                                                        |
| PCR                     | 5   | Program Clock Reference，节目时钟参考，用于恢复出与编码端一致的系统时序时钟 STC（System Time Clock）。 |
| stuffing_bytes          | x   | 填充字节，取值 `0xff`                                                                                  |

自适应区的长度要包含传输错误指示符标识的一个字节。pcr 是节目时钟参考，pcr、dts、pts 都是对同一个系统时钟的采样值，pcr 是递增的，因此可以将其设置为 dts 值，音频数据不需要 pcr。如果没有字段，ipad 是可以播放的，但 vlc 无法播放。打包 ts 流时 PAT 和 PMT 表是没有 adaptation field 的，不够的长度直接补 0xff 即可。视频流和音频流都需要加 adaptation field，通常加在一个帧的第一个 ts 包和最后一个 ts 包里，中间的 ts 包不加。

![](http://ww1.sinaimg.cn/large/8d56d744ly1g1jfvxth62j208g05ot8n.jpg)

**PAT 格式**

| 标志位                   | bit | 解释                                                                |
| ------------------------ | --- | ------------------------------------------------------------------- |
| table_id                 | 8   | PAT 表固定为 `0x00`                                                 |
| section_syntax_indicator | 1   | 固定为 `1`                                                          |
| zero                     | 1   | 固定为 `0`                                                          |
| reserved                 | 2   | 固定为 `11`                                                         |
| version_number           | 5   | 版本号，固定为 00000，如果 PAT 有变化则版本号加 1                   |
| current_next_indicator   | 1   | 固定为 `1`，表示这个 PAT 表可以用，如果为`0`则要等待下一个 PAT 表   |
| section_number           | 8   | 固定为 `0x00`                                                       |
| last_section_number      | 8   | 固定为 `0x00`                                                       |
| 开始循环                 |     |                                                                     |
| program_number           | 16  | 节目号为 `0x0000` 时表示这是 NIT，节目号为 `0x0001` 时,表示这是 PMT |
| reserved                 | 3   | 固定为 `111`                                                        |
| PID                      | 13  | 节目号对应内容的 PID 值                                             |
| 结束循环                 |     |                                                                     |
| CRC32                    | 32  | 前面数据的 CRC32 校验码                                             |

**PMT 格式**

| 标志位                   | bit | 解释                                                                                                                  |
| ------------------------ | --- | --------------------------------------------------------------------------------------------------------------------- |
| table_id                 | 8   | PMT 表取值随意 `0x02`                                                                                                 |
| section_syntax_indicator | 1   | 固定为 `0x01`                                                                                                         |
| zero                     | 1   | 固定为 `0x00`                                                                                                         |
| reserved_1               | 2   | 固定为 `0x03` (11)                                                                                                    |
| section_length           | 12  | 后面数据的长度                                                                                                        |
| program_number           | 16  | 频道号码，表示当前的 PMT 关联到的频道，取值 `0x0001`                                                                  |
| reserved_2               | 2   | 固定为 `0x03` (11)                                                                                                    |
| version_number           | 5   | 版本号，固定为 `00000`，如果 PAT 有变化则版本号加 1                                                                   |
| current_next_indicator   | 1   | 固定为 `0x01`                                                                                                         |
| section_number           | 8   | 固定为 `0x00`                                                                                                         |
| last_section_number      | 8   | 固定为 `0x00`                                                                                                         |
| reserved_3               | 3   | 固定为 `0x07` (111)                                                                                                   |
| PCR_PID                  | 13  | PCR(节目参考时钟)所在 TS 分组的 PID，指定为视频 PID；如果对于私有数据流的节目定义与 PCR 无关，这个域的值将为 0x1FFF。 |
| reserved_4               | 4   | 固定为 `0x0F` (1111)                                                                                                  |
| program_info_length      | 12  | 节目描述信息，指定为 `0x000` 表示没有                                                                                 |
| 开始循环 (std::vector)   |     |                                                                                                                       |
| stream_type              | 8   | 流类型，标志是 Video 还是 Audio 还是其他数据，h.264 编码对应 `0x1b`，aac 编码对应 `0x0f` ，mp3 编码对应 `0x03`        |
| reserved_5               | 3   | 固定为 `0x07` (111)                                                                                                   |
| elementary_PID           | 13  | 与 stream_type 对应的 PID                                                                                             |
| reserved_6               | 4   | 固定为 `0x0f` (1111)                                                                                                  |
| ES_info_length           | 12  | 描述信息，指定为 `0x000` 表示没有                                                                                     |
| 结束循环                 |     |                                                                                                                       |
| CRC32                    | 32  | 前面数据的 CRC32 校验码                                                                                               |

##### pes 层

pes 层是在每一个视频/音频帧上加入了时间戳等信息，pes 包内容很多，下面是一些最常用的。

![](http://ww1.sinaimg.cn/large/8d56d744ly1g1jh7ebc3hj20ao01sglf.jpg)

| 标志位            | bit | 解释                                                                          |
| ----------------- | --- | ----------------------------------------------------------------------------- |
| pes start code    | 3   | 开始码，固定为 0x000001                                                       |
| stream id         | 1   | 音频取值（0xc0-0xdf），通常为 `0xc0`<br/>视频取值（0xe0-0xef），通常为 `0xe0` |
| pes packet length | 2   | 后面 pes 数据的长度，0 表示长度不限制，<br/>只有视频数据长度会超过 `0xffff`   |
| flag              | 1   | 通常取值 `0x80`，表示数据不加密、无优先级、备份的数据                         |
| flag              | 1   | 取值 `0x80` 表示只含有 pts，取值 `0xc0` 表示含有 pts 和 dts                   |
| pes data length   | 1   | 后面数据的长度，取值 5 或 10                                                  |
| pts               | 5   | 33bit 值                                                                      |
| dts               | 5   | 33bit 值                                                                      |

pts 是显示时间戳、dts 是解码时间戳，视频数据两种时间戳都需要，音频数据的 pts 和 dts 相同，所以只需要 pts。有 pts 和 dts 两种时间戳是 B 帧引起的，I 帧和 P 帧的 pts 等于 dts。如果一个视频没有 B 帧，则 pts 永远和 dts 相同。从文件中顺序读取视频帧，取出的帧顺序和 dts 顺序相同。dts 算法比较简单，初始值 + 增量即可，pts 计算比较复杂，需要在 dts 的基础上加偏移量。

音频的 pes 中只有 pts（同 dts），视频的 I、P 帧两种时间戳都要有，视频 B 帧只要 pts（同 dts）。打包 pts 和 dts 就需要知道视频帧类型，但是通过容器格式我们是无法判断帧类型的，必须解析 h.264 内容才可以获取帧类型。

##### es 层

es 层指的就是音视频数据，这里只介绍 h.264 视频

打包 h.264 数据我们必须给视频数据加上一个 nalu (Network Abstraction Layer unit)，nalu 包括 `nalu header`和 `nalu type`。

`nalu header` 固定为 `0x00000001` (帧开始) 或 `0x000001` (帧中)。h.264 的数据是由 slice 组成的，slice 的内容包括： 视频、sps、pps 等。

`nalu type` 决定了后面的 h.264 数据内容。

```
+-------------------------------+
| 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
+---+---+---+---+---+---+---+---+
| F |  NRI  |       Type        |
+-------------------------------+
```

| 标志位 | bit | 解释                                                                                               |
| :----- | --- | -------------------------------------------------------------------------------------------------- |
| F      | 1   | forbidden_zero_bit，h.264 规定必须取 0                                                             |
| NRI    | 2   | nal_ref_idc，取值 0~3，指示这个 nalu 的重要性，I 帧、sps、pps 通常取 3，P 帧通常取 2，B 帧通常取 0 |
| Type   | 5   | nal_unit_type，参考下表                                                                            |

| nal_unit_type | 说明                            |
| ------------- | ------------------------------- |
| 0             | 未使用                          |
| **1**         | **非 IDR 图像片，IDR 指关键帧** |
| 2             | 片分区 A                        |
| 3             | 片分区 B                        |
| 4             | 片分区 C                        |
| **5**         | **IDR 图像片，即关键帧**        |
| **6**         | **SEI 补充增强信息单元**        |
| **7**         | **SPS 序列参数集**              |
| **8**         | **PPS 图像参数集**              |
| **9**         | **分解符**                      |
| 10            | 序列结束                        |
| 11            | 码流结束                        |
| 12            | 填充                            |
| 13 ~ 23       | 保留                            |
| 24 ~ 31       | 未使用                          |

加粗的类型内容是最常用的，打包 es 层数据时，pes 头和 es 数据之间要加入一个 `type=9` 的 `nalu`，关键帧 slice 前必须要加入 `type=7` 和 `type=8` 的 nalu，而且是紧邻。

![](http://ww1.sinaimg.cn/large/8d56d744ly1g1jf337jwrj20jk02w748.jpg)

### 使用 FFmpeg 对视频加密切片

加密算法有很多不同的类型，但 HLS 仅支持 AES-128。 AES 是分组密码的一个例子，它用固定大小的块，对数据进行加密，这是一种对称密钥算法，就是说加密解密用的是同样的密钥，AES-128 的密钥长度为 128 位。

HLS 在密码块链接（CBC）模式下使用 AES 。这意味着每个块都使用前一个块的密文进行加密，但对于第一块来说没有前一块，这时候就需要使用所谓的**初始化向量（IV）**。这个初始化向量也就是一个 16 字节的随机值，用于初始化加密过程。

在对视频加密之前，我们需要一把加密密钥，可以使用 OpenSSL 来创建：

```shell
openssl rand 16 > encrypt.key
```

这样就使用 OpenSSL 生成一个随机的 16 字节值，这对应于密钥长度（128 位），并保存到了 `encrypt.key` 文件中。

下一步是生成一个 IV。这一步是可选的。 （如果未提供任何值，则将使用段顺序号。）

```shell
> openssl rand -hex 16
2720c0161a8d052e6b0bf298409bca16
```

在使用 ffmpeg 加密视频的时候，我们需要告诉 ffmpeg 使用什么加密密钥，密钥的 URI 等等。我们使用 `-hls_key_info_file` 参数传递**密钥信息文件**的位置。

这个密钥信息文件，必须采用以下格式：

```
Key URI 密钥获取链接
Path to key file 密钥文件路径
IV (可选) 初始化向量
```

准备好了这个文件以后，cd 到要被加密的视频文件夹中，使用下面的命令对视频进行切片加密：

```shell
ffmpeg -y -i example.mp4 -hls_time 5 -hls_key_info_file key_info.key -hls_playlist_type vod -hls_segment_filename output/part_%04d.ts output/encrypted.m3u8
```

参数解释（FFmpeg 的参数详解请移步雷博的[CSDN 博客](https://blog.csdn.net/leixiaohua1020/article/details/12751349) —— 此处悼念雷博，天妒英才）：

- `-y` ，覆盖输出文件
- `-i` ，输入的视频文件名
- `-hls_time` ，ts 切片的视频时间长度
- `-hls_key_info_file` 密钥信息文件路径
- `-hls_playlist_type` ，输出的播放列表类型，vod 点播
- `-hls_segment_filename` ，定义输出 ts 片段的文件名格式，这里我输出到目录 `output` 下，会生成类似 `part_0001.ts` 的文件
- output/encrypted.m3u8 ， 这个是命令要输出的 m3u8 文件路径

执行完以后，会在当前目录生成一个 output 的文件夹，里面存放了切片加密后的 m3u8 文件和 ts 文件：

```she
➜ ll output
total 11136
-rw-r--r--  1 vimiix  staff   429B  3 25 14:58 encrypted.m3u8
-rw-r--r--  1 vimiix  staff   953K  3 25 14:58 part_0000.ts
-rw-r--r--  1 vimiix  staff   221K  3 25 14:58 part_0001.ts
-rw-r--r--  1 vimiix  staff   375K  3 25 14:58 part_0002.ts
-rw-r--r--  1 vimiix  staff   385K  3 25 14:58 part_0003.ts
```
