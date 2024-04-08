---
title: "python对属性的搜索优先级"
publish_time: "2023-07-28"
hidden: false
---
当获取元素的时候，实际上调用的是`object.__getattribute__(key)`
所谓的搜索优先级实际上是`object.__getattribute__(key)`实现的

Data Descriptor和Non-data Descriptor的不同体现在关于实例字典条目的覆盖和计算顺序上。
如果实例字典中包含了与Data Descriptor同名的属性，那么Data Descriptor优先。
如果实例字典中包含了与Non-data Descriptor同名的属性，实例字典优先。  

同时定义__get__()和__set__()方法，并且__set__()在调用时抛出AttributeError异常，就可以创建一个只读的Data Descriptor。
只需要定义一个抛出异常的__set__()方法就足以让该对象成为Data Descriptor。

```python
class demo(object):# 非数据描述符
    def __init__(self, *args, **kwargs):
        self.args = args
        self.kwargs = kwargs

    def __get__(self, instance, owner):
        return owner.__dict__[type(self).__name__](instance, *self.args, **self.kwargs)

class Descriptor(object):
    def __get__(self, instance, owner):
        return 'Descriptor'
    def __set__(self, instance, value):
        pass
class FOO(object):
    def __getattr__(self, item):
        return '__getattr__'
    def __init__(self, *args, **kwargs):
        pass
    # attr = Descriptor()
    demo = demo()
    # def demo(self):
    #    pass
    def __del__(self):
        pass
foo = FOO(1, 2, 3)

# foo.attr = 'instance attr'
# FOO.attr = 'class attr'
# type(foo).__dict__['attr'].__get__(foo, type(foo))
# FOO.__dict__['attr'].__get__(None, FOO)
print foo.attr
print FOO.attr

print FOO.__dict__['demo'] # <function demo at 0x02F67B30>
print FOO.demo # <unbound method FOO.demo>
print foo.demo # <bound method FOO.demo of <__main__.FOO object at 0x02F69E30>>

print FOO.__dict__['demo'].__get__(None, FOO) == FOO.demo #True
print type(foo).__dict__['demo'].__get__(foo, type(foo)) == foo.demo # #True


```

```python
def __getattribute__(self, key):
    # self是一个类
    if type(type) == type(self):
        pass
        # 和实例类似，区别1，但是没有__getattr__2，针对非数据描述符的处理方式是不同的。（unbound method）
    # self是一个实例
    else:
        # 数据型描述符
        _attr = type(self).__dict__.get(key)
        if _attr and hasattr(_attr, '__get__') and hasattr(_attr, '__set__'):
            _value = type(self).__dict__[key]
            if hasattr(_value, '__get__'):
                return _value.__get__(self, type(self))
            return _value
        del _attr
        # 实例属性
        _attr = self.__dict__.get(key)
        if _attr:
            return _attr
        del _attr
        # 非数据描述符
        _attr = type(self).__dict__.get(key)
        if _attr and hasattr(_attr, '__get__'):
            _value = type(self).__dict__[key]
            if hasattr(_value, '__get__'):
                return _value.__get__(self, type(self))
            return _value
        del _attr
        # 类属性
        _attr = type(self).__dict__.get(key)
        if _attr:
            return _attr
        # 父类属性
        for FClass in self.__class__.__mro__:
            _attr = FClass.__dict__.get(key)
            if _attr:
                return _attr
        # __getattr__
        if hasattr(self, '__getattr__'):
            return self.__getattr__(key)
        else:
            raise AttributeError('type object "{1}" has no attribute "{2}"'.format(str(self.__class__), key))
```
