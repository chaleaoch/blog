---
title: "一个不带插件的flask应用4 -- flask-filter"
publish_time: "2024-06-04"
hidden: false
---


<p style="color: rgba(127, 127, 127, 0.9);">由于接触的第一个web框架是django和django-rest-framework, 体会到了django-filter的妙处, 所以在开发flask项目的时候, 想自己做一个<p>

本filter基于[Peewee](https://github.com/coleifer/peewee), 但是可以很容易的扩展到sqlalchemy.

filter的组成部分:

1. filter_fields -- 因为字段的类型不同, 需要做一组对象做转换,校验和过滤
2. filter_class
   1. OrderingFilter -- 排序
   2. SearchFilter -- 搜索

## OrderingFilter

1. 获取默认的排序字段
2. 解析url中的排序参数
3. 校验排序字段是否合法
4. `return query.order_by()`

## SearchFilter

1. 校验搜索字段是否合法
2. 将搜索字段转换为peewee field
3. 将peewee field转换为filter field
4. 校验搜索值是否合法
5. 持续迭代搜索字段并 `query.where(<peewee field> <operator> <request field value>)`

具体代码请参考: [https://github.com/chaleaoch/flask-filter](https://github.com/chaleaoch/flask-filter)

<完>
