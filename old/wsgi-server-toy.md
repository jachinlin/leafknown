---
layout:     post
title:      实现一个简易的 wsgi 服务器
author:     JX
pub_date: 2016-09-01
tags:
    - socket
---

[上文](/old/wsgi-intro//)我们介绍了WSGI协议，本文，我们则亲自造一轮字：实现一个`wsgi server`。我们在模块`simple_wsgi`实现全部功能，主要包括：

- `WSGIServer`类：实现`wigs server`功能
- `make_serve`r函数：一个`wsgi server`工厂函数，用于创建 `WSGIServer`实例。

### WSGIServer

`WSGIServer`类包括一下属性和方法：

- 数据属性：
  - `request_queue_size`：类属性，请求连接队列大小
  - `listen_socket`：监听套接字
  - `server_name`：服务器名
  - `port`：服务器端口号
  - `client_connection`：连接套接字
  - `application`：web 应用
  - `headers_set`：响应头部信息
  - `request_data`：请求报文
  - `request_method`：请求方法
  - `path`：请求路径
  - `request_version`：HTTP版本号

- 方法

  - `__init__`：构造函数
  - `serve_forever`：启动服务
  - `_parse_request`：解析请求
  - `_get_environ`：构造environ
  - `_start_response`：构造start_responese函数
  - `set_app`：调用web application
  - `_handle_one_request`：处理请求
  - `_finish_response`：发送HTTP响应

  ​

接下来，我们来看看每个方法是怎么实现的。

#### \_\_init\_\_

```python
def __init__(self, server_address):
    # 创建监听套接字
    self.listen_socket = listen_socket = socket.socket(
        socket.AF_INET,
        socket.SOCK_STREAM
    )
    # 允许地址重用
    listen_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    # 绑定本地端点
    listen_socket.bind(server_address)
    # 被动监听
    listen_socket.listen(self.request_queue_size)
    # 获取服务器IP地址和端口号
    host, port = self.listen_socket.getsockname()[:2]
    self.server_name = socket.getfqdn(host)
    self.server_port = port
    # 响应头部信息
    self.headers_set = []
    # 其它实例属性
    self.application = None
    self.client_connection = None
    self.request_data = None
    self.request_method = None
    self.path = None
    self.request_version = None
```



#### set_app

```python
def set_app(self, application):
    """
    设置web应用
    :param application:
    :return:
    """
    self.application = application
```



#### serve_forever

```python
def serve_forever(self):
    listen_socket = self.listen_socket
    while True:
        # 连接套接字
        self.client_connection, client_address = listen_socket.accept()
        # 处理请求
        self._handle_one_request()
```



#### _parse_request

```python
def _parse_request(self, text):
    """
    解析请求行
    :param text: http请求报文
    :return:
    """
    # 获取请求行
    request_line = text.splitlines()[0]
    request_line = request_line.rstrip('\r\n')
    # 获取请求方法、请求URL、Http版本号
    (self.request_method,
     self.path,
     self.request_version
     ) = request_line.split()
```



#### _get_environ

```python
def _get_environ(self):
    """
    构建 environ
    :return:
    """
    env = dict()
    # WSGI 变量
    env['wsgi.version'] = (1, 0)
    env['wsgi.url_scheme'] = 'http'
    env['wsgi.input'] = StringIO.StringIO(self.request_data)
    env['wsgi.errors'] = sys.stderr
    env['wsgi.multithread'] = False
    env['wsgi.multiprocess'] = False
    env['wsgi.run_once'] = False
    # CGI 变量
    env['REQUEST_METHOD'] = self.request_method
    env['PATH_INFO'] = self.path
    env['SERVER_NAME'] = self.server_name
    env['SERVER_PORT'] = str(self.server_port)
    return env
```



#### _start_response

```python
def _start_response(self, status, response_headers, exc_info=None):
    # 添加必要的服务器头部信息
    server_headers = [
        ('Date', 'Tue, 31 Mar 2015 12:54:48 GMT'),
        ('Server', 'WSGIServer 0.2'),
    ]
    self.headers_set = [status, response_headers + server_headers]

```



#### _handle_one_request

```python
def _handle_one_request(self):
    """
    处理请求
    :return:
    """
    # 接受数据
    self.request_data = request_data = self.client_connection.recv(1024)
    # 解析请求行
    self._parse_request(request_data)
    # 构建环境变量
    env = self._get_environ()

    # 处理请求，获取响应报文体
    result = self.application(env, self._start_response)

    # 构建响应报文
    self._finish_response(result)
```



#### _finish_response

```python
def _finish_response(self, result):
    try:
        status, response_headers = self.headers_set
        response = 'HTTP/1.1 {status}\r\n'.format(status=status)
        for header in response_headers:
            response += '{0}: {1}\r\n'.format(*header)
        response += '\r\n'
        for data in result:
            response += data

        self.client_connection.sendall(response)
    finally:
        self.client_connection.close()
```



###make_server

`make_server`比较简单，接受一个端点地址和webapplication，返回一个server对象。

```python
def make_server(server_address, application):
    server = WSGIServer(server_address)
    server.set_app(application)
    return server
```



### 使用

现在，我们使用刚才完成的`simple_wsgi`和`flask`	来开发一个网络应用。代码如下：

```python
# app.py
# coding=utf-8

from simple_wsgi import make_server, SERVER_ADDRESS
from flask import Flask

app = Flask(__name__)

@app.route('/', methods=['GET', 'POST'])
def home():
    return '<h1>Home</h1>'

@app.route('/user/<name>')
def user(name):
    return '<h1>Hello, %s!</h1>' % name

if __name__ == '__main__':
    httpd = make_server(SERVER_ADDRESS, app)
    httpd.serve_forever()

```

这是效果：

![post_wsgi](/../img/post_wsgi.png)

![post_wsgi2](/../img/post_wsgi2.png)

It works! 我们居然开发出一个`wsgi server`！
