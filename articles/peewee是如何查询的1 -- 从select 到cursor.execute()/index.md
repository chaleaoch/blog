---
title: "peewee是如何查询的1 -- 从select 到database.execute()"
publish_time: "2025-07-11"
updates:
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">最近在工作中遇到一个 bug, 需要调查一下 peewee 的数据库连接实现, 这里简单记录一下分析过程, 方便日后查阅.</p>

版本信息:

```
peewee == 3.17.6
psycopg2 == 2.9.9   
```

Model.py 示例:

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
请注意, 我们只分析查询, 不分析查询后的结果如何返回, 也就是说我们要看到 `cursor.execute(sql, params or ())` 的调用, 而不是 `cursor.fetchall()`.

## 前置知识储备: AModel 的父类 (`peewee.Model`) 的元类 (`ModelBase`) 会加载 `db` 对象

执行顺序如下:

```python
class AModel(peewee.Model):
# AModel 的父类 peewee.Model --> peewee.py 6645
class Model(with_metaclass(ModelBase, Node)):
# peewee.Model 的元类 ModelBase.__new__ --> peewee.py 6547
cls._meta = Meta(cls, **meta_options)
```

这里的 `cls` 就是 `AModel`, `Meta` 具体内容暂且不表, 关键在于 `meta_options`, 它是一个字典结构, 包含了上面例子中的 `database = db`. 也就是说, `cls._meta.database == db`, 这是后续的关键.

## `AModel.select()` 会返回一个 `ModelSelect` 对象

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

将"执行权杖"移交给 database

```python
def _execute(self, database): # peewee.py 2278
    if self._cursor_wrapper is None:
        cursor = database.execute(self) # 这里的 database 就是之前的 db, 由此进入 database 的流程.
        self._cursor_wrapper = self._get_cursor_wrapper(cursor)
    return self._cursor_wrapper
```

## 总结

整理一下, 代码的执行顺序如下:

1. AModel.select()  返回 ModelSelect 对象
2. list(AModel.select()) # 调用 ModelSelect 对象的 __iter__ 方法
3. 调用ModelSelect 的 execute 方法
4. 调用ModelSelect 的 _execute 方法
5. 调用database.execute(self)

至于`database.execute(self)` 具体做了什么, 明天再继续分析.
<完>
