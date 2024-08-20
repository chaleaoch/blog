---
title: "cors扫盲"
publish_time: "2024-08-20"
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">扫盲,科普,笔记<p>

## 同源

协议/ip/端口号 满足一个就算跨域

## 行为

当跨域的时候,浏览器会阻止js获取到数据.

## 解决方案

### 简单请求

#### 定义

get/head/post 同时 header 限制在 Accept/Accept-Language/Content-Language/Last-Event-ID 同时 Content-Type：只限于三个值:

1. application/x-www-form-urlencoded
2. multipart/form-data
3. text/plain

#### 解决

浏览器自动增加一个header
Origin: <http://api.bob.com>
服务器根据这个值，决定是否同意这次请求。
如果没有Access-Control-Allow-Origin 浏览器会拦截这个请求.
如果有, 通常服务端会返回 四条header

```
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: FooBar
Content-Type: text/html; charset=utf-8


Access-Control-Allow-Origin 必选, *或者是 origin
Access-Control-Allow-Credentials 可选, 是否允许携带 **origin**的cookie, 如果为True, 则Access-Control-Allow-Origin 不能是*
Access-Control-Expose-Headers 可选, js从浏览器获取的response 只能默认获取六个header Cache-Control/Content-Language/Content-Type/Expires/Last-Modified/Pragma
```

### 非简单请求

首先发送预检请求 option

```txt
OPTIONS /cors HTTP/1.1
Origin: http://api.bob.com
Access-Control-Request-Method: PUT // 必选
Access-Control-Request-Headers: X-Custom-Header // 一个逗号分隔字符串
Host: api.alice.com
Accept-Language: en-US
Connection: keep-alive
User-Agent: Mozilla/5.0...
```

收到回应

```txt
HTTP/1.1 200 OK
Date: Mon, 01 Dec 2008 01:15:39 GMT
Server: Apache/2.0.61 (Unix)
Access-Control-Allow-Origin: http://api.bob.com // 这里
Access-Control-Allow-Methods: GET, POST, PUT // 这里, 所有方法
Access-Control-Allow-Headers: X-Custom-Header // 这里, 所有允许头, 已避免多次option请求
Access-Control-Allow-Credentials: true // 这里
Access-Control-Max-Age: 1728000 // 这里 表示有效期, 有效期内不需要再次发送options请求
Content-Type: text/html; charset=utf-8
Content-Encoding: gzip
Content-Length: 0
Keep-Alive: timeout=2, max=100
Connection: Keep-Alive
Content-Type: text/plain
```

预检通过之后, 后续请求规则同[[#简单请求]]
