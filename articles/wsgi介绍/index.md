---
title: "wsgi介绍"
publish_time: "2023-07-28"
hidden: false
---

*犹豫最近在组内推广我写的基于flask的框架, 可能会被问到一些基于flask的相对底层的问题. 所以重新学习了一下flask/wsgi和asgi. 趁此机会整理一个小系列出来, 这是第一篇.*

wsgi是一个软件层面的接口(协议).
解决server(gunicorn)和web application(flask)之间的通信问题.
wsgi分成三个部分, 服务端, 中间件和客户端. 其中中间件是可选的.

## 普通的web service

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

## WSGI具体内容

### 客户端

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

### 服务器

可以参考gunicorn的源码
<https://github.com/benoitc/gunicorn/blob/master/gunicorn/workers/sync.py>
commit id: 5b68c17b170c7b021a7a982a06c08e3898a5a640
`respiter = self.wsgi(environ, resp.start_response)`

```python
def handle_request(self, listener, req, client, addr):
    environ = {}
    resp = None
    try:
        ...
        respiter = self.wsgi(environ, resp.start_response)
        try:
            if isinstance(respiter, environ['wsgi.file_wrapper']):
                resp.write_file(respiter)
            else:
                for item in respiter:
                    resp.write(item)
            resp.close()
        finally:
            request_time = datetime.now() - request_start
            self.log.access(resp, req, environ, request_time)
            if hasattr(respiter, "close"):
                respiter.close()
```

### 中间件

中间件不是必须的, 在gunicorn和flask的组合中, 默认是没有中间件的.
当初制定wsgi协议的时候, 作者一开始的想法是将app设计成类似插件的形式, 每个app都非常轻, 通过中间件的方式, 将多个app组合在一起.
最后发展成现在这个样子.
中间件的格式很简单, 实现客户端的同时调用(另一个)客户端.

举个例子, `def __call__(` 这部分是客户端, `return self.app(environ, start_response)` 这部分是**调用**客户端.

```python
from your_application import app
from your_application.admin import app as admin_app

class DispatcherMiddleware:
    def __init__(
        self,
        app: WSGIApplication,
        mounts: dict[str, WSGIApplication] | None = None,
    ) -> None:
        self.app = app
        self.mounts = mounts or {}

    def __call__(
        self, environ: WSGIEnvironment, start_response: StartResponse
    ) -> t.Iterable[bytes]:
        script = environ.get("PATH_INFO", "")
        path_info = ""

        while "/" in script:
            if script in self.mounts:
                app = self.mounts[script]
                break

            script, last_item = script.rsplit("/", 1)
            path_info = f"/{last_item}{path_info}"
        else:
            app = self.mounts.get(script, self.app)

        original_script_name = environ.get("SCRIPT_NAME", "")
        environ["SCRIPT_NAME"] = original_script_name + script
        environ["PATH_INFO"] = path_info
        return app(environ, start_response)

application = DispatcherMiddleware(app, {'/admin': admin_app})
```
