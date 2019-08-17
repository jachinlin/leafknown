---
title:      Odoo Routing
author:     JX
pub_date: 2017-07-21
tags:
    - Odoo
summary: Odoo routing 模块源码解析。
---

前置知识：

- 元编程
- 装饰器
- 函数属性
- inspect模块（自省）
- defaultdict

`Odoo`中所有的web请求都是由库 [werkzeug](http://werkzeug.pocoo.org) 驱动的，`Odoo`在` werkzeug`的基础上进一步封装，隐藏了`werkzeug`的复杂性，让我们更加方便地定义路由。探索隐藏在`Odoo`带来的便利背后的工作原理将是一件有趣的体验，现在，就让我们来体验这种乐趣吧。

首先，我们先看看`Odoo`是如何定义一个路由的。

```python
# controllers/main.py

from odoo import http
from odoo.http import request


class Main(http.Controller):
    @http.route('/hello', type='http', auth='none')
    def hello(self):
        return "<h1>hello world!</h1>"
```

可以看到，`Odoo`的路由实现方式和`Flask`的实现方式相似，相对于`werkzeug`简单便利多了。

重启服务器后，访问`/hello`我们就能看到熟悉的`hello world!`了。

![odoo url](/../img/odoo url.png)

Odoo的路由定义有两个重要的部分：

- 我们定义的控制器`controller`继承了`odoo.http.Controller`
- 视图函数`view_func`由`odoo.http.route`装饰着

我们先来看看`odoo.http.route`，为了方便讨论，我们暂时忽略`route`里边的`type`和`auth`参数。

```python
# part of odoo/http.py
def route(route=None, **kw):
    routing = kw.copy()
    assert 'type' not in routing or routing['type'] in ("http", "json")
    def decorator(f):
        if route:
            if isinstance(route, list):
                routes = route
            else:
                routes = [route]
            routing['routes'] = routes
        @functools.wraps(f)
        def response_wrap(*args, **kw):
            response = f(*args, **kw)
            if isinstance(response, Response) or f.routing_type == 'json':
                return response

            if isinstance(response, basestring):
                return Response(response)

            if isinstance(response, werkzeug.exceptions.HTTPException):
                response = response.get_response(request.httprequest.environ)
            if isinstance(response, werkzeug.wrappers.BaseResponse):
                response = Response.force_type(response)
                response.set_default()
                return response

            _logger.warn("<function %s.%s> returns an invalid response type for an http request" % (f.__module__, f.__name__))
            return response
        response_wrap.routing = routing
        response_wrap.original_func = f
        return response_wrap
    return decorator
```

可以看出，`route`是个装饰器，这个装饰器主要做了两件事

- 定义了一个新的视图函数，这个函数和我们之前的视图函数效果一样，不过会对视图函数的返回值进行一些必要的转换（这也是为了简化我们的视图函数）
- 将原视图函数和`route`的参数绑定到新视图函数上

由于装饰器这个语法糖，新的视图函数和原视图函数的名字和作用都是一样的，不过新视图函数多了一个返回值自动处理过程和两个函数属性。（这里是一个很好的运用函数属性的例子，把函数当作一个名字空间。）

现在，我们来看看`odoo.http.Controller`

```python
# part of odoo/http.py
class Controller(object):
    __metaclass__ = ControllerType
```

这个类啥都没实现，就指定了一个元类，重要的东西还是在元类里实现的。

```python
# part of odoo/http.py

controllers_per_module = collections.defaultdict(list)

class ControllerType(type):
    def __init__(cls, name, bases, attrs):
        super(ControllerType, cls).__init__(name, bases, attrs)

        # 将请求类型设置在原视图函数上(原视图函数也是个命名空间)
        for k, v in attrs.items():
            if inspect.isfunction(v) and hasattr(v, 'original_func'):
                routing_type = v.routing.get('type')
                v.original_func.routing_type = routing_type

        # 将控制器存储在controllers_per_module中
        name_class = ("%s.%s" % (cls.__module__, cls.__name__), cls)
        class_path = name_class[0].split(".")
        if not class_path[:2] == ["odoo", "addons"]:
            module = ""
        else:
            module = class_path[2]
        controllers_per_module[module].append(name_class)

 # 注：这里省略了关于类继承的处理
```

我们自定义的`controller`继承`odoo.http.Controller`，`odoo.http.Controller`的元类为`ControllerType`，也就是说，自定义`controller`是由`odoo.http.Controller`生成的。`odoo.http.Controller`在生成自定义`controller`后会进行一些操作

- 将请求类型（也就是`route`的`type`参数的值）保存到原视图函数的属性`routing_type`中。

  > 这里有个疑问，为什么不打这一步放到装饰器`route`中呢，毕竟route中有`response_wrap.routing = routing`，即把url以及各种参数保存到新视图函数的属性`routing`上。我觉得把请求类型绑定到原视图函数这一步放到route中，更能体现代码的一致性。

- 第二是将控制器`controller`的信息保存到`controllers_per_module`中，`controllers_per_module`是一个字典，key是模块名（odoo主要就是由一个个业务模块构成的），value是一个列表，列表的元素是一个tuple，tuple的一个元素是控制器controller的名字（包括路径名的字符串），第二个参数是控制器controller本身。`controllers_per_module`的具体形式可以参考下图：

![controllers_per_module](/../img/controllers_per_module.png)

至此，关于`odoo.http.route`和`odor.http.Controller`的研读已完，但我们的脚步不应停止，因为我们还没见到一点`werkzeug`的影子。目前，我们只是发现ODOO将控制器保存到`controllers_per_module`而已。

在`http.py`中，我们发现 Odoo web application 分发请求时会调用`routing_map`函数。

```python
# part of odoo/http.py

def routing_map(modules, nodb_only, converters=None):
  	# 生成 Map 实例
    routing_map = werkzeug.routing.Map(strict_slashes=False, converters=converters)
    for module in modules:
        if module not in controllers_per_module:
            continue
        for _, cls in controllers_per_module[module]:
            o = cls()
            members = inspect.getmembers(o, inspect.ismethod)
            for _, mv in members:
                if hasattr(mv, 'routing'):
                  	# 请求类型默认为http,权限默认为user
                    routing = dict(type='http', auth='user', methods=None, routes=None)
                    routing.update(mv.routing)
                    # 这里是权限判断，我们暂时认为其值为`True`即可
                    if not nodb_only or routing['auth'] == "none":
                        assert routing['routes'], "Method %r has not route defined" % mv
                        # 构建endpoint
                        endpoint = EndPoint(mv, routing)
                        for url in routing['routes']:
                            xtra_keys = 'defaults subdomain build_only strict_slashes redirect_to alias host'.split()
                            kw = {k: routing[k] for k in xtra_keys if k in routing}
                            # 添加rule
                            routing_map.add(werkzeug.routing.Rule(url, endpoint=endpoint, methods=routing['methods'], **kw))
    return routing_map

# 注：为了便于分析，这里同样删除了跟控制器继承相关的代码
```

`routing_map`这个函数是比较简单的，（说明：   第一个参数表示目标模块列表。odoo是按模块分割业务功能的，我们需要什么功能就加载什么模块，而`controllers_per_module`中存有所有模块的控制器信息，所以这里需要`modules`这个参数），就是生成一个`werkzeug.routing.Map`对象，然后遍历`modules`，从`controllers_per_module`找到响应路由信息，生成`werkzeug.routing.Rule`对象，添加到Map对象中。



The end.
