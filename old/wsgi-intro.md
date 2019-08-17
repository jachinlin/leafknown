---
title:      WSGI 初探
author:     JX
pub_date:   2016-08-27
tags:
    - wsgi
---


web服务器需要接收HTTP请求、解析HTTP请求以及发送HTTP响应，处理这些底层操作是十分耗时的，且容易出错。为了提高web服务的开发效率，人们定义了一个接口——`WSGI` (Web Server Gateway Interfere)——将web服务器分成了两部分：

- web 服务器：也称为`wsgi server`
- web 应用：也称为`wsgi application`

web服务器需要根据语境才能区分具体是指广义的服务器还是狭义的服务器。`wsgi server`主要负责处理TCP连接、处理HTTP原始请求以及发送HTTP响应；而`wsgi application`则专注于web业务。其他编程语言也拥有类似的接口：例如Java的`Servlet API`和Ruby的`Rack`。



![wsgi](/../img/wsgi.jpg)



### wsgi application

`WSGI`接口非常简单，它要求`wsgi application` 实现一个函数`application`。这个函数接受两个参数

- `environ`


- `start_response`

这两个参数由`wsgi server`传递给`application`。`environ`是一个包含了所有HTTP请求信息的字典；`start_response`是一个函数，需要在`applicataion`中调用，它负责将HTTP响应状态码以及头部发送到`wsgi server`。有了WSGI接口，`application`只需要：

1. 从`environ`获取HTTP请求信息
2. 根据不同的请求信息构建不同的响应body
3. 通过`start_response`向wsgi发送响应状态码、header
4. 返回响应body。

而不用关心处理TCP连接等底层代码。

下面就是一个极简application：

```python
# client.py
def application(environ, start_response):
  # parse the environ
  start_response('200 OK', [('Content-Type', 'text/html')])
  return ['<h1>Hello Web!</h1>']

```

其实，`application`也可以是类、实现了`__call__`方法的类实例等可调用对象。在大多数web框架中，`application`是通过类实例表示的。

### wsgi server

wsgi server 则负责提供environ和start_response参数，并调用application；以及实现处理TCP连接、处理HTTP原始请求、发送HTTP响应等底层逻辑。

常用的 wsgi server 有` Gunicorn` 和 `Tornado`。Python的 `wsgiref` 模块也内置了一个 wsgi server ，该服务器完全符合 WSGI 协议，但性能低下，只能用于开发和测试。下面，我们就 wsgiref 模块来看 wsgi server 是如何加载 application 的：

```python
# server.py
from wsgiref.simple_server import make_server
from client import application

# 创建一个服务器，IP地址为空，端口是8000，处理函数是application:
httpd = make_server('', 8000, application)
# 开始监听HTTP请求:
httpd.serve_forever()
```

一个简单的web服务就这么生成了。启动程序，在浏览器地址栏输入`http://localhost:8000/`就能看到咱们的劳动成果了。
