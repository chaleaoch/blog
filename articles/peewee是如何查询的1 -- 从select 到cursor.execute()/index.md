---
title: "peewee是如何查询的1 -- 从select 到database.execute()"
publish_time: "2025-07-11"
updates:
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">最近在工作中遇到一个bug, 需要调查一下peewee的数据库连接情况, 简单记录一下, 过几天又忘了<p>

版本:  

```
peewee == 3.17.6
psycopg2 == 2.9.9   
```

Model.py:  

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

入口是 `list(AModel.select())`, 让我们来看看peewee是如何利用pg的cursor来查询数据的.

## AModel的父类(`peewee.Model`)的元类(`ModelBase`)会加载`db`对象

按照顺序执行

```python
class AModel(peewee.Model):
# AModel的父类peewee.Model --> peewee.py 6645`
class Model(with_metaclass(ModelBase, Node)): 
# peewee.Model的元类ModelBase.__new__ --> peewee.py 6547
cls._meta = Meta(cls, **meta_options)
```

这里的cls就是`AModel`, 至于`Meta`是啥不重要, 重要的是`meta_options`, 它是一个字典结构, 包含上面例子中的`database = db`, 也就是说, `cls._meta.database == db` 这里是关键, 后面要考.

## `AModel.select()`会返回一个`ModelSelect`对象

```python
class ModelSelect(BaseModelSelect, Select): # peewee.py 7339
    def __init__(self, model, fields_or_models, is_default=False): # model 是 AModel
        self.model = self._join_ctx = model
        self._joins = {}
        self._is_default = is_default
        fields = _normalize_model_select(fields_or_models)
        super(ModelSelect, self).__init__([model], fields)
```

当`list(AModel.select())`的时候, `ModelSelect`对象对象的`__iter__`方法会被调用, 这个方法会调用`self.execute()`来执行查询.

```python
def __iter__(self): #peewee.py 7273 # self 是 ModelSelect对象
    if not self._cursor_wrapper:
        self.execute()
    return iter(self._cursor_wrapper)
```

这里调用方没有传参, 但是因为装饰器`database_required`的原因, 将database的参数传递给了`execute`方法.

```python
@database_required # peewee.py 2105
def execute(self, database): # self 是 ModelSelect对象
    return self._execute(database)
# ...
def database_required(method): # peewee.py 2029
    @wraps(method)
    def inner(self, database=None, *args, **kwargs): # self 是 ModelSelect对象
        database = self._database if database is None else database # 这里传递进去的
        if not database:
            raise InterfaceError('Query must be bound to a database in order '
                                 'to call "%s".' % method.__name__)
        return method(self, database, *args, **kwargs) # 这个 method 就是 self.execute()
    return inner

```

最后, 还剩下一个问题, `self._database`是如何被赋值的?

```python
class _ModelQueryHelper(object): # 这个类是ModelSelect的父类
    default_row_type = ROW.MODEL

    def __init__(self, *args, **kwargs):
        super(_ModelQueryHelper, self).__init__(*args, **kwargs)
        if not self._database:
            self._database = self.model._meta.database # peewee.py 7212
```

如果前面的内容你都跟上了, 就可以推导出:
`self.model._meta.database` 就是之前的`db`, 也就是`AModel`的`Meta`类中的`database = db`

整理一下, 前面的代码执行顺序:

```python
AModel.select() # 返回 ModelSelect 对象
list(AModel.select()) # 调用 ModelSelect 对象的 __iter__ 方法
# 进入 ModelSelect 的 execute 方法
self.execute() # 进入 ModelSelect 的 execute 方法
# 进入 ModelSelect 的 _execute 方法
self._execute(database) # 进入 ModelSelect 的 _execute 方法
```

## 将"权杖"移交给database

```python
def _execute(self, database): # peewee.py 2278
    if self._cursor_wrapper is None:
        cursor = database.execute(self) # 这里的database就是之前的db, 由此进入database时间.
        self._cursor_wrapper = self._get_cursor_wrapper(cursor)
    return self._cursor_wrapper
```

`database.execute(self)` 都做了什么, 明天再说吧.

<完>
