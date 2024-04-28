---
title: "wsgi和asgi"
publish_time: "2023-07-28"
hidden: true
---

## WSGI

是一个软件层面的接口(协议).
解决server(gunicorn)和web application(flask)之间的通信问题.

### 普通的web service

在不考虑wsgi的情况下,

```python
import socket

EOL1 = b"\n\n"
EOL2 = b"\n\r\n"
body = """<html>Hello, world!<html>"""
response_prams = [
    "HTTP/1.0 200 OK",
    "Date: Sat, 27 Jun 2028 13:00:00 GMT",
    "Content-Type: text/html; charset=utf-8",
    "Content-Length: {}".format(len(body)),
    "",
    body,
]
response = "\r\n".join(response_prams)


def handle_connection(conn, addr):
    request = b""
    while EOL1 not in request and EOL2 not in request:
        request += conn.recv(1024)
    print(request)
    conn.send(response.encode())
    conn.close()


def main():
    serversocket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    serversocket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    serversocket.bind(("127.0.0.1", 8000))
    serversocket.listen(5)
    print("http://127.0.0.1:8000")
    try:
        while True:
            conn, addr = serversocket.accept()
            handle_connection(conn, addr)
    except KeyboardInterrupt:
        serversocket.close()


if __name__ == "__main__":
    main()
```

### WSGI具体内容

#### 客户端

```python
def application(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/html')])
    return [b'<h1>Hello, web!</h1>']
```

environ, 字典, 保存http相关的信息
<https://peps.python.org/pep-3333/#environ-variables>

```python
{
    'REQUEST_METHOD': 'GET',
    'PATH_INFO': '/index.html',
    'QUERY_STRING': '',
    'CONTENT_TYPE': '',
    'CONTENT_LENGTH': '',
}
```

start_response 回调,两个参数, 第一个参数和状态码相关, 第二个参数和header相关
`start_response('200 OK', [('Content-Type', 'text/html')])`

返回值, 是一个可迭代对象, 里面是**字节串**, 内容是body.

#### 服务器

阅读gunicorn源码, 解释如何调用application

#### 中间件

中间件不是必须的, 在gunicorn和flask的组合中, 默认是没有中间件的.
当初制定wsgi协议的时候, 作者一开始的想法是将app设计成类似插件的形式, 每个app都非常轻, 通过中间件的方式, 将多个app组合在一起.
最后发展成现在这个样子.
中间件的格式很简单, 实现客户端的同时调用(另一个)客户端.

举个例子

```python
from werkzeug.wsgi import DispatcherMiddleware
from your_application import app
from your_application.admin import app as admin_app

application = DispatcherMiddleware(app, {'/admin': admin_app})
```
