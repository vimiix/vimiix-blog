---
title: '使用SQLAlchemy以多IP方式连接openGauss数据库'
date: 2021-07-08 23:38:48
categories: 'golang'
tags:
  - 'SQLAlchemy'
  - 'Python'
  - 'openGauss'
---

### 前置声明

>  由于 [openGauss](https://opengauss.org/zh/) 数据库本身也开源不久，所以周边基础设施也正处于遍地开花的阶段，所以本文不保证长期的时效性，仅针对现阶段的问题，提出一种解决方案。

## openGauss 介绍

按照官网的介绍，openGauss 是一款高性能，高安全，高可靠的开源关系型数据库管理系统，采用木兰宽松许可证v2发行。openGauss内核早期源自开源数据库PostgreSQL，融合了华为在数据库领域多年的内核经验，在架构、事务、存储引擎、优化器及ARM架构上进行了适配与优化。

openGauss 在2020年6月30日开放源代码，代码托管在 gitee 上。

目前我所在公司也主要是做数据库方面的事情，且也基于 openGauss 内核研发了一款商业版的数据库 [MogDB](https://enmotech.com/products/MogDB)，感兴趣的也可以去了解一下。

## 背景

针对 openGauss 的基础设施不完善，我之前基于 [py-postgresql](https://github.com/python-postgres/fe) 1.3.0 版本，开发了 openGauss 的 python 驱动，并提交到了 openGauss 官方仓库：https://gitee.com/opengauss/openGauss-connector-python-pyog

该驱动适用于不使用 ORM，以SQL形式交互的 python 程序。

但是在对接客户的过程中，客户提出了这样的需求：

- 程序中用到了 SQLAlchemy 做为ORM来和数据库交互
- 连接数据库时，需要支持类似 JDBC 多 IP 的方式连接（例如：`user:password@host1:port1,host2:port2/db` 这种形式），这种需求主要是用于后端数据库部署形态为主备架构，且没有固定的虚拟IP用于连接，所以需要将主备所有机器的IP都传递进去，但只能返回主库的连接。

赶紧去扫了一下 SQLAlchemy 的源码，SQLAlchemy 中目前支持的 pg 驱动有：

- [psycopg2](https://docs.sqlalchemy.org/en/14/dialects/postgresql.html#module-sqlalchemy.dialects.postgresql.psycopg2)
- [pg8000](https://docs.sqlalchemy.org/en/14/dialects/postgresql.html#module-sqlalchemy.dialects.postgresql.pg8000)
- [asyncpg](https://docs.sqlalchemy.org/en/14/dialects/postgresql.html#module-sqlalchemy.dialects.postgresql.asyncpg)
- [psycopg2cffi](https://docs.sqlalchemy.org/en/14/dialects/postgresql.html#module-sqlalchemy.dialects.postgresql.psycopg2cffi)
- [py-postgresql](https://docs.sqlalchemy.org/en/14/dialects/postgresql.html#module-sqlalchemy.dialects.postgresql.pypostgresql)
- [pygresql](https://docs.sqlalchemy.org/en/14/dialects/postgresql.html#module-sqlalchemy.dialects.postgresql.pygresql)

其实只要在驱动里面实现了 [PEP-249](https://www.python.org/dev/peps/pep-0249/) (DBAPI 2.0 接口规范) 就可以。但是我们都知道，在使用  SQLAlchemy 创建 engine 的时候，需要在 `://` 前面使用固定的字符串进行驱动的选择。比如 `postgresql+pypostgresql://...`，内部其实就是把加号 replace 成点，然后去加载对应的驱动模块文件。

但是 openGauss 的驱动可能是目前不支持的，所以我们需要解决的就是两个问题：

- 开发一个驱动，能够实现 DBAPI 2.0 接口规范，且支持多IP连接
- 增强 SQLAlchemy ，支持我们开发的 openGauss 驱动

解决了以上两个问题后，就可以和平时使用 SQLAlchemy 一样来操作 openGauss 了。

### 完整架构图

<img src="https://static.vimiix.com/upic/2021-07-08/WeChat665cbcb9d30440234880c049693b01ed.png" alt="完整架构图" style="zoom:67%;" />

## py-opengauss 库

第一个问题就是要开发一个 openGauss 的驱动，我们上面提到我之前基于 [py-postgresql](https://github.com/python-postgres/fe) 1.3.0 开发了一个驱动，但是当时仅支持了 openGauss 的连接和SQL交互，现在需要基于目前的需求进行开发。

> 代码目前已经发布到 github 上，源码地址：https://github.com/vimiix/py-opengauss
>
> 包已经推送到 pypi 上，最新版本 1.3.6

首先是 DBAPI 2.0 接口，庆幸的是  py-postgresql 本身就已经支持了 DBAPI 2.0 接口规范，代码位于 `postgresql.driver.dbapi20`。

所以我们需要做的是如何在接口上支持多IP。

为了减少对于现有代码的侵入，我修改了 dbapi20 模块中 [ `connect` ](https://github.com/vimiix/py-opengauss/blob/6cb1b6ba46a9ad86f091f5b397454dac82ed622e/py_opengauss/driver/dbapi20.py#L416) 函数，上层需要通过 `host` 字段将IP部分（包含所有主机和端口号），传递进来进行处理。

例如： `{'host': 'host1:5432,host2:6432'}` 

在 connect 函数中，会遍历去进行连接，直到找到主库的连接后停止遍历返回连接。

这样就解决了第一个问题。

这部分功能已经发布到 1.3.6 版本中，所以在 pip 安装的时候需指定版本：

```bash
pip install py-opengauss==1.3.6
```

## SQLAlchemy 定制 

接下来是第二个问题，需要增强 SQLAlchemy 支持 py-opengauss 驱动。

> 定制后的代码地址：https://github.com/vimiix/sqlalchemy

我 fork 了一份 SQLAlchemy 的代码，在现有的代码上，在 `lib/sqlalchemy/dialects/postgresql` 中新增 pyopengauss 的模块，在内部 dbapi 接口处返回我们的驱动即可：

```python
@classmethod
def dbapi(cls):
  from py_opengauss.driver import dbapi20
  return dbapi20
```

这样我们就在可以 create_engine 的时候，使用 `postgresql+pyopengauss://` 这个前缀来进行连接了。

此时还存在另外一个问题，我们在业务层创建 engine 时传入的连接字符串假如是多IP形式的，SQLAlchemy 内部会通过一个固定的[正则表达式](https://github.com/sqlalchemy/sqlalchemy/blob/990069b2e8627b7c7c649d1198390ec728b43089/lib/sqlalchemy/engine/url.py#L700)来匹配 `user`, `password`, `host`, `port`,`db` 以及 `query` 。但是对于多IP的字符串，匹配出来以后仅会匹配到第一个 `host` 部分，剩余部分被匹配到了 `port` 部分，类似下面这样：

<img src="https://static.vimiix.com/upic/2021-07-08/WeChat8b3e7a354445dda961402c8102d1a308.png" style="zoom:50%;" />

这样的话，对于 port 字段进行 `int(port)` 转换整型的时候，必然会报错。

所以我对于正则表达式进行了[修改](https://github.com/vimiix/sqlalchemy/blob/49b7c05cd4c65e7d9967b8664aadae38cd2e9acb/lib/sqlalchemy/engine/url.py#L700)，让 `host:port` 部分先不拆开，一起匹配出来， 然后我们手动进行解析。如果是多IP的情况，就不进行拆分，直接放入 `host` 字段传递给驱动（上面提到过，驱动部分我们也是通过该字段解析的）。如果不是多IP形式，就直接 split 拆分，`host` 字段放 host， `port` 字段放 port ，和常规连接一样提交给驱动即可。

具体代码实现：

```python
ipv4host = components.pop("ipv4host")
        ipv6host = components.pop("ipv6host")
        if ipv4host:
            if ',' in ipv4host:
                # multiple host
                components["host"] = ipv4host
            elif ':' in ipv4host:
                host, port = ipv4host.split(':', 1)
                components["host"] = host
                components["port"] = int(port)
            else:
                components["host"] = ipv4host
        else:
            components["host"] = ipv6host
```

进行了多IP的适配以后，此时的 SQLAlchemy  就能够满足我们的需求了。

由于我们是基于 SQLAlchemy  的定制，且为了业务代码的少变更，所以安装方式只能是通过下载源码，然后手动通过 `python setup.py install` 来进行安装。安装后，业务层仅需要修改一个连接字符串即可。

## 使用方式

1. 安装 py-opengauss

- 通过 pip 安装

```bash
pip install py-opengauss==1.3.6
```

- 通过源码安装

```bash
git clone -b for-orm https://github.com/vimiix/py-opengauss.git
cd py-opengauss
python3 setup.py install
```

2. 安装定制后的 SQLAlchemy 

```bash
git clone https://github.com/vimiix/sqlalchemy
cd sqlalchemy
python3 setup.py install
```

3. 业务层修改连接串

```python
from sqlalchemy import create_engine
engine = create_engine('postgresql+pyopengauss://user:password@host1:port1,host2:port2/db')
```

