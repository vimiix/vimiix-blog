---
title: '[Python]python如何方便的操作MySQL和Oracle数据库'
date: 2018-01-30 01:27:48
categories: 'Python'
tags: [Python, 'pymysql', 'cx_Oracle']
---

# 环境

- python3 [官方网站下载](https://www.python.org/downloads/release/python-364/)
- pymysql `pip3 install pymysql`
- cx_Oracle `pip3 install cx-Oracle`

<!--more-->

# 使用 pymysql 创建 MySQL 操作用的实例类和游标类

> gist 地址：[https://gist.github.com/vimiix/c5611283604968cc8d038781e2fdd982](https://gist.github.com/vimiix/c5611283604968cc8d038781e2fdd982)

```python
class MySQLCursor:
	"""创建一个游标类"""

    def __init__(self,cursor,logger):
        self.cursor=cursor
        self.logger=logger

    def execute(self,sql,params=None):
        self.logger.info(sql+str(params))
        self.cursor.execute(sql, params)
        self.cursor.execute("select last_insert_id()")
        res = self.cursor.fetchone()
        if len(res)==1:
            if type(res)==type({}):
                return res['last_insert_id()']
            if type(res)==type(()):
                return res[0]
        return  0

    def query(self,sql,params=None,with_description=False):
        self.logger.info(sql+str(params))
        self.cursor.execute(sql, params)
        rows = self.cursor.fetchall()
        if with_description:
            res = rows, self.cursor.description
        else:
            res = rows
        return res

class MySQLInstance:
	"""创建一个实例类"""

    def __init__(self,host,port,username,password,schema=None,
                 charset='utf8',dict_result=False,logger=None):
        self.host=host
        self.port=port
        self.username=username
        self.password=password
        self.schema=schema
        self.charset=charset
        self.dict_result=dict_result
        if logger is None:
            logger=logging.getLogger(__name__)
        self.logger=logger

    def __enter__(self):
        self.con=pymysql.connect(host=self.host,port=self.port,user=self.username,
                                 password=self.password,charset=self.charset,db=self.schema)
        if self.dict_result:
            self.cursor=self.con.cursor(pymysql.cursors.DictCursor)
        else:
            self.cursor=self.con.cursor()
        return MySQLCursor(self.cursor,self.logger)
    def __exit__(self,exc_type, exc_value, exc_tb):
        self.cursor.execute("commit")
        self.cursor.close()
        self.con.close()

```

有了这两个类，在操作数据库的时候就可以结合 Python 中的 `with` 关键词优雅的使用了。

`dict_result` 为 `True` 时，返回结果集为字典，`False` 为元祖

示例：

```python
mysql_db = {
	"host": '11.11.11.11', # ip 地址
	'port': 3306,          # 端口
	'username': 'root',    # 用户名
	"password": 'xxxx',    # 密码
	'schema':'db'          # 数据库名
}

with MySQLInstance(**mysql_db, dict_result=True) as db:
    print db.query('select 1')
```

# 使用 cx_Oracle 创建 MySQL 操作用的实例类和游标类

> gist 地址：[https://gist.github.com/vimiix/92e5f37b38a35aaf9b9f3c0e7bb00538](https://gist.github.com/vimiix/92e5f37b38a35aaf9b9f3c0e7bb00538)

```python
class OracleCursor:
	"""游标类"""

    def __init__(self, cursor, logger, dict_result):
        self.cursor = cursor
        self.logger = logger
        self.dict_result = dict_result

    def _dict_result(self, cursor):
        cols = [d[0].lower() for d in cursor.description]
        def row_factory(*args):
            return dict(zip(cols, args))
        return row_factory

    def execute(self, sql, params=None):
        if params:
            self.cursor.execute(sql, params)
        else:
            self.cursor.execute(sql)

    def query(self, sql, params=None, with_description=False):
        if params:
            self.cursor.execute(sql, params)
        else:
            self.cursor.execute(sql)
        if self.dict_result:
            self.cursor.rowfactory = self._dict_result(self.cursor)
        rows = self.cursor.fetchall()
        if with_description:
            res = rows, self.cursor.description
        else:
            res = rows
        return res


class OracleInstance:
	"""实例类"""

    def __init__(self, username, password, host, port, sid,
                 charset='utf8',dict_result=False,logger=None):
        self.username = username
        self.password = password
        self.host = host
        self.port = port
        self.sid = sid
        self.charset = charset
        self.dict_result = dict_result
        if logger is None:
            logger=logging.getLogger(__name__)
        self.logger = logger

    def __enter__(self):
        dsn_tns = cx_Oracle.makedsn(self.host, self.port, self.sid)
        self.con = cx_Oracle.connect(self.username, self.password, dsn_tns)
        self.cursor = self.con.cursor()
        return OracleCursor(self.cursor, self.logger, self.dict_result)

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.cursor.execute("commit")
        self.cursor.close()
        self.con.close()

```

和 MySQL 同样用法, 示例：

```python
ORACLE_DB = {
    'username': 'sys',       # 用户名
    'password': 'xxxxx',     # 密码
    'host': '11.11.11.11',   # ip地址
    'port': 1521,            # 端口号
    'sid': 'orcl'            # sid
}

with OracleInstance(**ORACLE_DB, dict_result=True) as db:
    print(db.query('select * from dba_tables'))

```
