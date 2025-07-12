---
title: "peewee是如何查询的"
publish_time: "2025-07-11"
updates:
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">最近在工作中遇到一个 bug, 需要深入了解 peewee 的数据库连接实现. 本文记录了我的分析过程, 方便日后查阅.</p>

版本信息:

```
peewee == 3.17.6
psycopg2 == 2.9.9   
```

## Model.py 示例

```python
db = peewee.PostgresqlDatabase(
    database=DB_DATABASE,
    user=DB_USER,
    password=DB_PASSWORD,
    host=DB_HOST,
    port=DB_PORT,
)

class AModel(peewee.Model):
    ... # 省略其他字段
    class Meta:
        database = db
```

入口是 `list(AModel.select())`
让我们来看看 peewee 是如何利用 pg 的 cursor 查询数据的.
本文只分析查询的执行过程, 不涉及结果的返回(即只关注 `cursor.execute(sql, params or ())` 的调用, 而非 `cursor.fetchall()`).

## 前置知识: AModel 的父类 (`peewee.Model`) 的元类 (`ModelBase`) 会加载 `db` 对象

```python
class AModel(peewee.Model):
# AModel 的父类 peewee.Model --> peewee.py 6645
class Model(with_metaclass(ModelBase, Node)):
# peewee.Model 的元类 ModelBase.__new__ --> peewee.py 6547
cls._meta = Meta(cls, **meta_options)
```

这里的 `cls` 就是 `AModel`, `Meta` 具体内容暂且不表, 关键在于 `meta_options`, 它是一个字典结构, 包含了上面例子中的 `database = db`. 也就是说, `cls._meta.database == db`, 后面会用到.

## 从 Model.Select 到 database.execute()

`AModel.select()` 会返回一个 `ModelSelect` 对象:

```python
class ModelSelect(BaseModelSelect, Select): # peewee.py 7339
    def __init__(self, model, fields_or_models, is_default=False): # model 是 AModel
        self.model = self._join_ctx = model
        self._joins = {}
        self._is_default = is_default
        fields = _normalize_model_select(fields_or_models)
        super(ModelSelect, self).__init__([model], fields)
```

当执行 `list(AModel.select())` 时, 会调用 `ModelSelect` 对象的 `__iter__` 方法, 该方法内部会调用 `self.execute()` 来执行查询:

```python
def __iter__(self): # peewee.py 7273, self 是 ModelSelect 对象
    if not self._cursor_wrapper:
        self.execute()
    return iter(self._cursor_wrapper)
```

虽然调用方没有传参, 但由于装饰器 `database_required` 的作用, `database` 参数会自动传递给 `execute` 方法.

```python
@database_required # peewee.py 2105
def execute(self, database): # self 是 ModelSelect 对象
    return self._execute(database)
# ...
def database_required(method): # peewee.py 2029
    @wraps(method)
    def inner(self, database=None, *args, **kwargs): # self 是 ModelSelect 对象
        database = self._database if database is None else database # 这里传递进去的
        if not database:
            raise InterfaceError('Query must be bound to a database in order '
                                 'to call "%s".' % method.__name__)
        return method(self, database, *args, **kwargs) # 这个 method 就是 self.execute()
    return inner
```

那么, `self._database` 是如何被赋值的?

```python
class _ModelQueryHelper(object): # 这个类是 ModelSelect 的父类
    default_row_type = ROW.MODEL

    def __init__(self, *args, **kwargs):
        super(_ModelQueryHelper, self).__init__(*args, **kwargs)
        if not self._database:
            self._database = self.model._meta.database # peewee.py 7212
```

结合前面的内容可以推导出:
`self.model._meta.database` 就是之前的 `db`, 也就是 `AModel` 的 `Meta` 类中的 `database = db`.

### "执行权杖"的传递: 交给 `database`

```python
def _execute(self, database): # peewee.py 2278
    if self._cursor_wrapper is None:
        cursor = database.execute(self) # 这里的 database 就是之前的 db, 由此进入 database 的流程.
        self._cursor_wrapper = self._get_cursor_wrapper(cursor)
    return self._cursor_wrapper
```

整理一下, 代码的执行顺序如下:

1. `AModel.select()`  返回 `ModelSelect` 对象
2. `list(AModel.select())` 调用 `ModelSelect` 对象的 `__iter__` 方法
3. 调用 `ModelSelect` 的 `execute` 方法
4. 调用 `ModelSelect` 的 `_execute` 方法
5. 调用 `database.execute(self)`

## 从 `database.execute(self)` 到 `cursor.execute(sql, params or ())`

到目前为止, 这里的 `database` 特指 `PostgresqlDatabase`, 实际上 peewee 提供了多种数据库连接对象和 `Mixin`, 后面会有说明.

```python
def execute(self, query, commit=None, **context_options):
    if commit is not None:
        __deprecated__('"commit" has been deprecated and is a no-op.')
    ctx = self.get_sql_context(**context_options)
    sql, params = ctx.sql(query).query() # 这里生成sql和参数
    return self.execute_sql(sql, params) # 执行sql
```

这里的 self 是 PostgresqlDatabase 的父类 Database, 接下来的一系列方法调用都在 Database 中, 一路跳转即可. 为节约篇幅, 这里用伪代码演示:

```txt
execute_sql --> cursor = self.cursor() --> {
    if self.is_closed() and auto_connect: # 数据库连接关闭且自动连接
        self.connect() # 这里做了什么?
    return self._state.conn.cursor() # self._state 是什么?
}
```

继续看 self.connect() 做了什么:

```txt
self.connect --> {
    if not self._state.closed: # 如果已打开
        if reuse_if_open: # 且允许重用
            return False # 直接返回, 不会重新建立连接
        raise OperationalError('Connection already opened.') # 否则抛出异常
    self._state.reset() # self._state 是什么?
    self._state.set_connection(self._connect()) # self._connect() 由子类实现, 这里的子类是 PostgresqlDatabase
    ...
    self._initialize_connection(self._state.conn) # 这是一个回调, peewee 本身没有具体实现, 应用层可自定义, 后面会有举例
}
```

父类(Database)的 `_connect` 是一个抽象方法, 必须由子类实现. 这里是 PostgresqlDatabase._connect()

```txt
def _connect(self):
    params = self.connect_params.copy() # 实例化PostgresqlDatabase时传入的参数
    ...
    conn = psycopg2.connect(**params) # 建立连接
    ...
    return conn # 返回连接
```

### 流程总结

1. 获取 cursor
   1. 调用 `self.connect()`
      1. 检查连接状态, 如果已连接且不允许重用, 则抛出异常
      2. 如果已连接且允许重用, 则直接返回, 不重连
      3. 如果未连接, 调用 psycopg2.connect(**params) 建立连接
2. 利用 cursor 执行 sql

整体流程大致如此, 但这里还留下一些疑问:

### self._state

self._state 是什么?
self._state 默认是一个 ThreadLocal 对象, 用于存储当前线程的数据库连接状态信息. 这里的 `self._state.conn` 就是通过 set_connection 建立的连接.
`self._state = _ConnectionLocal()`

也就是说, 这个连接状态只在当前线程有效. 如果是多线程场景, 每个线程都会建立一个新的连接.

self._state.closed 在哪里赋值?
只有手动关闭连接(db.close())时, self._state.closed 会被设置为 True.
只有调用 self._state.set_connection 时, self._state.closed 会被设置为 False.
结论: 连接建立后, closed 永远是 False, 也就是说, 这个连接永远不会自动重连. 当出现网络抖动或数据库服务端因超时等原因断开连接时, 执行查询语句会报错!

## 实现断开自动重连

用数据库连接池(PooledPostgresqlDatabase)是否可以实现断开自动重连? 答案是不行. PooledPostgresqlDatabase 虽然重写了 connect 和 _connect 方法, 超过 stale_timeout 后会自动关闭连接, 但如果连接被动关闭且没有客户端超时, 最本质的 self._state.closed 依然是 False, 最终查询依然会报异常.
PooledPostgresqlDatabase 的真正目的是防止高并发场景下连接耗尽, 而不是实现自动重连.

```python
def _connect(self):
    while True:
        try:
            # Remove the oldest connection from the heap.
            ts, _, c_conn = heapq.heappop(self._connections) # 从连接池中获取连接
            conn = c_conn
        except IndexError:
            ts = conn = None
            logger.debug('No connection available in pool.')
            break
        else:
            if self._is_closed(conn): # 如果这个连接已关闭
                ts = conn = None
            elif self._stale_timeout and self._is_stale(ts): # 如果超时
                self._close(conn, True) # 超时关闭
                ts = conn = None
            else:
                break
    if conn is None: # 没能从池子里找到可用连接
        if self._max_connections and (
                len(self._in_use) >= self._max_connections): # 连接池已满
            raise MaxConnectionsExceeded('Exceeded maximum connections.')
        conn = super(PooledDatabase, self)._connect() # 新建连接
        ...
    self._in_use[key] = PoolConnection(ts, conn, time.time()) # 放入池子
    return conn
```

最终的 MySQL 解决方案是 ReconnectMixin.
ReconnectMixin 重写了 execute_sql, 包裹了执行查询时的异常. 如果捕获到特定异常, 则尝试重连数据库.

```python
def execute_sql(self, sql, params=None, commit=None):
    ...
    return self._reconnect(super(ReconnectMixin, self).execute_sql, sql, params)

def _reconnect(self, func, *args, **kwargs):
    try:
        return func(*args, **kwargs)
    except Exception as exc:
        # 如果在事务中, 不自动重连, 直接抛出异常
        if self.in_transaction():
            raise exc
        ...
        if not self.is_closed():
            self.close()
            self.connect() # 重连

        return func(*args, **kwargs) # 再次执行
```

很遗憾, 针对PostgreSQL, peewee 官方并没有类似的自动重连机制, 可能是因为 PostgreSQL 本身没有 idle 过长自动断开的机制. 最后我手动实现了一个简化版, 供参考:

```python
import time

import peewee
from playhouse.pool import PooledPostgresqlDatabase


class ReconnectMixin:
    def execute_sql(self, sql, params=None, commit=None):
        try:
            super(ReconnectMixin, self).execute_sql(sql, params)
        except Exception as e:
            if self.in_transaction():
                raise e
        if not self.is_closed():
            self.close()
            self.connect()

        return super(ReconnectMixin, self).execute_sql(sql, params)


class MyDb(ReconnectMixin, PooledPostgresqlDatabase):
    pass


db = MyDb(
    database="db",
    user="usr",
    password="passwd",
    host="localhost",
    port=5432,
    max_connections=20,
    stale_timeout=300,
)


class AModel(peewee.Model):
    id = peewee.AutoField(primary_key=True)
    name = peewee.CharField(max_length=255)

    class Meta:
        database = db


while True:
    try:
        print(list(AModel.select()))
        time.sleep(1)
    except Exception as e:
        time.sleep(1)
        print(f"Error: {e}")
```

最终实现的效果:  
![xwig9z.gif](https://files.catbox.moe/xwig9z.gif)

## 最后一个例子
>
> self._initialize_connection(self._state.conn) # 这是一个回调, peewee 本身没有具体实现, 应用层可自定义, 后面会有举例

这里举一个例子, DBA不给我们统一的timezone, 导致我们写入的datetime数据不对, 需要每次建立连接的时候, 执行一个一次性的SQL `SET TIME ZONE 'UTC'`

```python
class MyPooledPostgresqlDatabase(PooledPostgresqlDatabase):
    def _initialize_connection(self, conn):
        with conn.cursor() as cur:
            cur.execute("SET TIME ZONE 'UTC';")
        return conn
```

<完>
