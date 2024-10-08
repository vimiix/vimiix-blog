---
title: 'psycopg2基于openGauss中execPramasBatch和execPreparedBatch接口测试'
date: 2023-08-04 21:50:48
categories: 'python'
tags:
  - 'driver'
  - 'psycopg2'
  - 'benchmark'
---

> 如果要查看 Psycopg2 不同接口之间批量操作对比测试，请访问[这篇笔记](https://www.vimiix.com/posts/2023-07-12-psycopg2-batch-api-benchmark/)

## 测试环境

| 组件   | 说明                      |
|--------|---------------------------|
| 客户端操作系统 | Rocky Linux 8    |
|服务端配置| 2C6G, 40G HDD|
| CPU | Intel Xeon Processor (Icelake) |
| 数据库 | MogDB 5.0    |
| 网络   | 300M宽带     |
| Python | 3.6.8       |
| Psycopg2 | [vimiix/openGauss-connector-python-psycopg2](https://gitee.com/vimiix/openGauss-connector-python-psycopg2/tree/dev) |

> 注意：
>
> 1. 由于真正的性能和服务器的配置，网络情况相关性也比较大，本测试所有的测试用例环境条件一致，只有参数作为变量，所以请不要注重数值本身，重点关注不同情况下的性能比例
> 2. 本测试只取了 100/1000/10000 这个page_size，具有一定的性能趋势，但不代表一味的增大 page_size 就可以提高性能，必然存在一个性能拐点的参数值，而且不同的场景存在不同的性能拐点，要找到性能拐点仍需根据实际情况进行更多的测试。

<!--more-->

## 插入 INSERT

### execute_values

| rows\page_size | 100(default) | 1000 | 10000 |
|---------|-----------|---------|--------------|
| 10,000  |    2089   |    349     |   204    |
| 50,000  |   10842   |    1801     |   707     |
| 100,000 |   21445  |    3625     |    1257     |

![image.png](https://static.vimiix.com/upic/2024-09-20/iNwbLv.png)

### execute_params_batch

| rows\page_size | 100(default) | 1000 | 10000 |
|---------|-----------|---------|-------------|
| 10,000  |   2243    |    504     |    312    |
| 50,000  |   11574   |    2591     |   1539     |
| 100,000 |   22552  |     5511    |    2980     |

![image.png](https://static.vimiix.com/upic/2024-09-20/748CoC.png)

### execute_prepared_batch

|rows\page_size| 100(default) | 1000       | 10000    |
|---------|-------------------|-------------|--------------|
| 10,000  |  2328            |   506      |     339     |
| 50,000  |  11307           |   2836      |  1480      |
| 100,000 |  22976           |   5920      |  3425       |

![image.png](https://static.vimiix.com/upic/2024-09-20/yeQRc8.png)

### 相同行数时，不同 page_size 的接口性能对比

![image.png](https://static.vimiix.com/upic/2024-09-20/OQynhS.png)
![image.png](https://static.vimiix.com/upic/2024-09-20/PXguIq.png)

## 更新 UPDATE

### execute_values

> 用法比较饶，需要把 values 的部分单独用一个 %s 占位

| rows\page_size | 100(default) | 1000 | 10000 |
|---------|-----------|---------|--------------|
| 10,000  |    3884   |     562     |   393    |
| 50,000  |   15367   |    3429     |   2601     |
| 100,000 |   25882  |    11313     |    5356     |

![image.png](https://static.vimiix.com/upic/2024-09-20/roeISI.png)

### execute_params_batch

| rows\page_size | 100(default) | 1000 | 10000 |
|---------|-----------|---------|-------------|
| 10,000  |   2746    |    810     |    579    |
| 50,000  |   13359   |    4478     |   3026     |
| 100,000 |   28432  |     8529    |    5923     |

![image.png](https://static.vimiix.com/upic/2024-09-20/iPGUHF.png)

### execute_prepared_batch

|rows\page_size| 100(default) | 1000       | 10000    |
|---------|-------------------|-------------|--------------|
| 10,000  |  2747            |   896      |     635     |
| 50,000  |  12860           |   6153      |  3626      |
| 100,000 |  28445           |   10103      |  10828       |

![image.png](https://static.vimiix.com/upic/2024-09-20/c64pUD.png)

## 相同行数时，不同 page_size 的接口性能对比

![image.png](https://static.vimiix.com/upic/2024-09-20/pFengW.png)
![image.png](https://static.vimiix.com/upic/2024-09-20/cWneQ2.png)

## 结论

1. 任何接口，在对于大批量操作时，调大 page_size 均可以得到有效的性能提升
2. 从数据来看， execute_values 的性能不管在INSERT还是UPDATE的表现，都稍稍好于 execute_params_batch 和 execute_prepared_batch（由于测试的局限性，可能也不一定）。execute_params_batch 和 execute_prepared_batch 两个接口性能差别不大，底层走的是一套接口。
3. 从使用上来说，execute_values 使用比较饶，如果要使用这个接口更新，写SQL的复杂度大于其他两个，比如相同的更新操作：
    - execute_values： `UPDATE t3 as t SET j = data.j FROM (VALUES %s) AS data (j, i) WHERE t.i = data.i`
    - 其他两个：`UPDATE t3 SET j=$1 WHERE t3.i=$2`
