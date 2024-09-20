---
title: 'psycopg2 批量操作接口性能测试'
date: 2023-07-12 14:50:48
categories: 'python'
tags:
  - 'driver'
  - 'psycopg2'
  - 'benchmark'
---


## 测试版本

本测试基于 openGauss 版本的 psycopg2 驱动

```python
import psycopg2 as pg
>>> pg.__libpq_version__
90204
>>> pg.__version__
'2.8.6 (dt dec pq3 ext)'
```

## 测试环境

| 组件   | 说明                      |
|--------|---------------------------|
| 客户端 | Rocky Linux 8 虚拟机      |
| 数据库 | openGauss 3.0.3 in docker |
| 网络   | 本地回路网卡              |
| Python  | 3.6.8              |

<!--more-->

## 测试接口

| 接口名                                                                                                                                                                   | 说明                                                                                                                                   | 备注                                                                                                              |
|:-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| [cursor.executemany(query, vars_list)](https://www.psycopg.org/docs/cursor.html#cursor.executemany)                                                                      | 执行一个数据库操作，vars_list 列表中的所有参数会逐个被应用到query 中，每组参数都会单独封包发送给服务端。                                   | 该函数主要用于更新数据库的命令，查询返回的任何结果集都将被丢弃。在其当前实现中，此方法并不比在循环中执行execute()快。 |
| [psycopg2.extras.execute_batch(cur, sql, argslist, page_size=100)](https://www.psycopg.org/docs/extras.html#psycopg2.extras.execute_batch)                               | 批量执行一个数据库操作，执行的SQL和 executemany 相同，只是单个数据包发送时会发送一批SQL，数量由page_size决定。这样可以减少和服务端的通信次数 | execute_batch()也可以和预处理语句(PREPARE, EXECUTE, DEALLOCATE)一起使用。                                           |
| extras.execute_batch + 预处理语句                                                                                                                                        | 使用PREPARE提交创建一个statement，然后通过 execute_batch 提交                                                                            |                                                                                                                   |
| [psycopg2.extras.execute_values(cur, sql, argslist, template=None, page_size=100, fetch=False)](https://www.psycopg.org/docs/extras.html#psycopg2.extras.execute_values) | 将参数和SQL封装为一条SQL执行，单条SQL中参数的个数由 page_size 决定。                                                                      |                                                                                                                   |

## 性能对比

### INSERT

#### 测试数据

| rows    | executemany | execute_batch | prepare+execute_batch | execute_values |
|---------|-------------|---------------|:----------------------|:---------------|
| 10,000  | 9.782       | 0.707         | 0.501       | 0.266          |
| 50,000  | 52.979      | 3.123         | 2.637        | 1.226         |
| 100,000 | 111.504    | 6.831         | 4.557         | 2.125         |

#### INSERT耗时对比图

![batch-insert](https://oss-emcsprod-public.modb.pro/image/editor/20230712-5709755a-dafa-4096-858c-7301e5160cd2.jpg)

#### INSERT 去除 executemany 对比

![batch-insert-exclude-executemany](https://oss-emcsprod-public.modb.pro/image/editor/20230712-fa992e73-e08f-4e7e-8be3-79ea905a339e.jpg)

### UPDATE

#### 测试数据

| rows    | executemany | execute_batch | prepare+execute_batch | execute_values |
|---------|-------------|---------------|:----------------------|:---------------|
| 10,000  | 5.015       | 0.617         | 0.425            | 0.356          |
| 50,000  | 24.639      | 3.467         | 1.905            | 5.237          |
| 100,000 | 52.095    | 6.927         | 3.473            | 21.102           |

#### UPDATE 耗时对比图

![16892490588494.jpg](https://oss-emcsprod-public.modb.pro/image/editor/20230713-75ea074d-253a-4f82-a94c-eeb3e2280727.jpg)

### DELETE

#### 测试数据

(*100000 条数据组耗时太久不做展示*)

| rows    | executemany | execute_batch | prepare+execute_batch | execute_values |
|---------|-------------|---------------|:----------------------|:---------------|
| 10,000  | 15.020       | 8.699         | 0.277     | 6.204          |
| 50,000  | 248.154      | 227.958         | 1.455   | 142.732        |

#### DELETE 耗时对比图

![16892491996399.jpg](https://oss-emcsprod-public.modb.pro/image/editor/20230713-f2675fd6-8e22-4b0b-b08b-564932a60d60.jpg)

## 性能分析

从耗时对比来看，插入、更新、删除在不同的数据量情况下性能是不同的，用户应该根据自己的业务场景来选择使用哪一种操作接口。

插入性能从低到高依次为：

`executemany < execute_batch < prepare+execute_batch < execute_values`

更新性能从低到高依次为：

`executemany < execute_values < execute_batch < prepare+execute_batch`

删除性能从低到高依次为：

`executemany < execute_batch < execute_values < prepare+execute_batch`

性能的高低主要是由于在向服务端发送数据包时的方式不同导致，下面以插入的SQL为例，通过 wireshark 进行抓包可以看出 psycopg2 在通信过程中不同批处理接口的封包情况。

### executemany

`executemany` 提交SQL的时候是逐个应用给的参数，每个SQL都单独发送给服务端

![fd9636479f4c.jpg](https://oss-emcsprod-public.modb.pro/image/editor/20230712-1b04e125-3c1f-42bf-8760-fd9636479f4c.jpg)

### execute_batch

`execute_batch` 接口区别于 `executemany` 的是，在发送给后端的单个请求包里的数据会一次性提交一批的SQL，这样可以减少和服务器之间通信的往返次数

![54901fdb310d.jpg](https://oss-emcsprod-public.modb.pro/image/editor/20230712-52a6a6c0-4fcb-494e-84ea-54901fdb310d.jpg)

### prepare+execute_batch

`prepare` 可以提前在数据库里面创建一个预备语句对象，在执行 prepare 语句的时候，指定的SQL已经经了解析、分析、重写，这样在后续执行 EXECUTE 时就避免了重复解析分析的工作，从而起到优化性能的作用。

![8d8abc70246d.jpg](https://oss-emcsprod-public.modb.pro/image/editor/20230712-efca89b2-a99f-4c4e-8fce-8d8abc70246d.jpg)

### execute_values

前面的三个接口，不管是单个提交还是批量提交，最终都是一行数据一个SQL发送到服务端的，所以服务端需要逐个执行，而 `execute_values` 接口是会按照 page_size 分组参数后，每组参数一次性组成一个SQL进行提交。

![b7d1-8031d10e389c.jpg](https://oss-emcsprod-public.modb.pro/image/editor/20230712-8fc8095f-6a20-4c01-b7d1-8031d10e389c.jpg)
![b7d1-8031d10e389c.jpg](https://oss-emcsprod-public.modb.pro/image/editor/20230712-8fc8095f-6a20-4c01-b7d1-8031d10e389c.jpg)

## 测试代码

执行方式：`python test.py <api> <row> <operation>`

- `<api>` 支持： `executemany`, `execute_batch`, `prepare`, `execute_values`
- `<operation>` 支持 `insert`, `update`, `delete`

```python
# coding: utf-8

# Usage: python test.py <api> <count> <operation>

import time
import sys
import psycopg2 as pg
from psycopg2.extras import execute_batch, execute_values
from contextlib import contextmanager

if sys.argv[3] == "insert":
    args = [[str(i), i] for i in range(int(sys.argv[2]))]
elif sys.argv[3] == "update":
    args = [[i, str(i)] for i in range(int(sys.argv[2]))]
elif sys.argv[3] == "delete":
    args = [[i] for i in range(int(sys.argv[2]))]
'''
- *dbname*: the database name
- *database*: the database name (only as keyword argument)
- *user*: user name used to authenticate
- *password*: password used to authenticate
- *host*: database host address (defaults to UNIX socket if not provided)
- *port*: connection port number (defaults to 5432 if not provided)
'''
conf = {
    'dbname': "postgres",
    'user': 'gaussdb',
    'password': '',
    'host': '',
    'port': 26000,
    'sslmode': 'disable'
}


@contextmanager
def calc_time(s):
    start = time.time()
    yield
    end = time.time()
    print(f"{s} of '{sys.argv[3]}' cost: ", end - start)


sql_map = {
    "insert": {
        1: "INSERT INTO t_psycopg2_benchmark VALUES (%s, %s)",
        2: "INSERT INTO t_psycopg2_benchmark VALUES ($1, $2)",
        3: "INSERT INTO t_psycopg2_benchmark VALUES %s",
    },
    "update": {
        1: "UPDATE t_psycopg2_benchmark as t SET f_value = %s WHERE t.f_key = %s",
        2: "UPDATE t_psycopg2_benchmark as t SET f_value = $1 WHERE t.f_key = $2",
        3: "UPDATE t_psycopg2_benchmark as t SET f_value = data.v1 FROM (VALUES %s) AS data (id, v1) WHERE t.f_key = data.id",
    },
    "delete": {
        1: "DELETE FROM t_psycopg2_benchmark as t WHERE t.f_key=%s",
        2: "DELETE FROM t_psycopg2_benchmark as t WHERE t.f_key=$1",
        3: "DELETE FROM t_psycopg2_benchmark as t WHERE t.f_key IN (%s)",
    }
}

def insert_data(conn):
    print("* preparing data ...")
    args = [[str(i), i] for i in range(int(sys.argv[2]))]
    cursor = conn.cursor()
    sql = "insert into t_psycopg2_benchmark values %s"
    execute_values(cursor, sql, args)
    conn.commit()


def main():
    try:
        conn = pg.connect(**conf)
        print("* connect success")
    except Exception as e:
        print(f"connect failed: {e}")
        return

    cursor = conn.cursor()

    sql = "drop table if exists t_psycopg2_benchmark"
    cursor.execute(sql)
    sql = "create table t_psycopg2_benchmark (f_key text primary key, f_value numeric)"
    cursor.execute(sql)

    api = sys.argv[1]
    if sys.argv[3] != "insert":
        insert_data(conn)

    print("* benchmarking ...")
    if api == "executemany":
        with calc_time("executemany"):
            sql = sql_map[sys.argv[3]][1]
            cursor.executemany(sql, args)
            conn.commit()
    elif api == "execute_batch":
        with calc_time("execute_batch"):
            sql = sql_map[sys.argv[3]][1]
            execute_batch(cursor, sql, args)
            conn.commit()
    elif api == "prepare":
        with calc_time("execute_values"):
            cursor.execute(f"PREPARE test_stmt AS {sql_map[sys.argv[3]][2]}")
            if sys.argv[3] == "delete":
                execute_batch(cursor, "EXECUTE test_stmt (%s)", args)
            else:
                execute_batch(cursor, "EXECUTE test_stmt (%s, %s)", args)
            cursor.execute("DEALLOCATE test_stmt")
            conn.commit()
    elif api == "execute_values":
        with calc_time("execute_values"):
            sql = sql_map[sys.argv[3]][3]
            execute_values(cursor, sql, args)
            conn.commit()
    else:
        print(f"unknow api: {api}")

    if sys.argv[3] != "delete":
        cursor.execute("delete from t_psycopg2_benchmark")
        conn.commit()


if __name__ == "__main__":
    main()
```
