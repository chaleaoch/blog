---
title: "future对象"
publish_time: "2024-05-13"
hidden: false
---
<p style="color: rgba(127, 127, 127, 0.9);">Future对象其实在平时的工作中使用的并不是很多, 最近在研究fastapi以及底层的asyncio, 里面遇到了future对象和task对象, 仔细看看.<p>

在python中, future可以指两个对象, 一个是[concurrent.futures.Future](https://docs.python.org/zh-cn/3/library/concurrent.futures.html), 一个是[asyncio.Future](https://docs.python.org/zh-cn/3/library/asyncio-future.html#asyncio.Future). 两者的api差别不大. 本文主要介绍前者.

future对象主要用于在同步环境下实现异步操作. 举个例子

```python
import time
from concurrent.futures import ThreadPoolExecutor
from datetime import datetime


def task(n):
    print(f"task:{datetime.now()}")
    return n * n


def async_run(func, *args):
    executor = ThreadPoolExecutor(max_workers=3)
    future = executor.submit(func, *args)
    executor.shutdown(wait=False)
    return future


f = async_run(task, 3)
time.sleep(3)  # do something business logic
print(f"main:{datetime.now()}")
print(f.result())
```

## Executor

一个抽象类, 实现类(ThreadPoolExecutor)可以参考上面的例子, 更常见的写法是使用with上下文管理自动的shutdown, 例子参考最下面`as_completed`的例子.

## Future

由`Executor.submit()` 返回

主要方法:
`result(timeout=None)` 阻塞直到结果返回, 或者超时, 如有异常, 正常触发.
`exception(timeout=None)` 阻塞直到结果返回, 或者超时, 如有异常, 返回异常对象, 否则, 返回None
`cancel()` 尝试取消任务, 如果任务已经开始, 则返回False, 否则返回True

## 函数

`wait(fs, timeout=None, return_when=ALL_COMPLETED)`

返回一个元组, 已经完成的future对象和未完成的future对象
timeout, 超时时间
return_when, 一个枚举值, ALL_COMPLETED, FIRST_COMPLETED, FIRST_EXCEPTION
![alt text](index/image.png)

```python
import concurrent.futures
import random
import time


def task(x):
    time.sleep(random.random())
    return x * x


with concurrent.futures.ThreadPoolExecutor() as executor:
    futures = []
    for i in range(10):
        future = executor.submit(task, i)
        futures.append(future)

    done, not_done = concurrent.futures.wait(futures, timeout=0.5)
    print(f"done:{len(done)}")  # done:6
    print(f"not_done:{len(not_done)}")  # not_done:4

    for future in done:
        print(future.result())  # 81 36 9 0 49 4

```

`concurrent.futures.as_completed(fs, timeout=None)`

返回一个迭代器, 迭代器中的元素都是运行完的future对象, 下面是官网给的一个例子.
其中with可以理解为一种快捷方式. 自动调用`executor.shutdown`

```python
import concurrent.futures
import urllib.request

URLS = ['http://www.foxnews.com/',
        'http://www.cnn.com/',
        'http://europe.wsj.com/',
        'http://www.bbc.co.uk/',
        'http://nonexistant-subdomain.python.org/']

# Retrieve a single page and report the URL and contents
def load_url(url, timeout):
    with urllib.request.urlopen(url, timeout=timeout) as conn:
        return conn.read()

# We can use a with statement to ensure threads are cleaned up promptly
with concurrent.futures.ThreadPoolExecutor(max_workers=5) as executor:
    # Start the load operations and mark each future with its URL
    future_to_url = {executor.submit(load_url, url, 60): url for url in URLS}
    for future in concurrent.futures.as_completed(future_to_url):
        url = future_to_url[future]
        try:
            data = future.result()
        except Exception as exc:
            print('%r generated an exception: %s' % (url, exc))
        else:
            print('%r page is %d bytes' % (url, len(data)))
```

全文完.
