---
title:      werkzeug —— Context Locals
author:     JX
pub_date: 2017-09-10
tags:
    - werkzeug
---

Python标准库中有一个叫做 `thread locals`的东西，一个local对象就是一个线程安全和线程隔离的全局变量。也就是说，不管你是往local对象写数据还是读数据，该local对象都会获取当前程序所处的线程，并提取该线程相应的数据。然而，这种做法有一些缺陷。除了多线程，我们还会使用其它的并发方式。一种很流行的做法就是使用`greenlets`(协程)。

`werkzeug.local`的出现就是为了解决这个问题的。`werkzeug.local`和`thread local`有着类似的功能，但对`greenlets`也起作用。直奔源码，看看这到底是怎么实现的：

### Local

```python
# part of werkzeug.local

class Local(object):
    __slots__ = ('__storage__', '__ident_func__')

    def __init__(self):
        object.__setattr__(self, '__storage__', {})
        object.__setattr__(self, '__ident_func__', get_ident)

    def __getattr__(self, name):
        try:
            return self.__storage__[self.__ident_func__()][name]
        except KeyError:
            raise AttributeError(name)

    def __setattr__(self, name, value):
        ident = self.__ident_func__()
        storage = self.__storage__
        try:
            storage[ident][name] = value
        except KeyError:
            storage[ident] = {name: value}
    # ......

```

Local有两个属性`__storage__`,`__ident_func__`。`__storage__`用来存储数；`__ident_func__`的值为`get_ident`，从下面代码可以看出该函数是用来获取当前线程ID或者greenlet对象。（简单起见，下文中线程、协程、greenlet对象均表示同一意思）

```python
try:
    from greenlet import getcurrent as get_ident
except ImportError:
    try:
        from thread import get_ident
    except ImportError:
        from _thread import get_ident
```

werkzeug.local实现线程数据安全的关键是重写了`__setattr__`和`__getattr__`这两个两个魔术方法。重写了这两个魔术方法之后。`l.some_attr`(`l=Local()`)就不再是它看上去那么简单了：

- 它首先调用`get_ident`方法获取当前线程／协程ID
- 然后对这个ID空间下的`some_attr`属性进行读写
- 也就是说`l.some_attr`相当于`l.__storage__[get_ident()].some_attr`

简单说，werkzeug就是在Local中构建一个内部字典`__storage__`用来保存线程资源，字典的key是线程／协程ID，字典value就是对应线程／协程的资源(资源也是以key-value对的字典形式表示)。但是直接通过线程ID这个key访问相应的资源就显得太丑了，werkzeug重写了`__setattr__`等魔术方法很巧妙地解决了这个问题。

werkzeug还定义`LocalStack`、`LocalProxy`等类，这些类在web框架设计中也很常用。

### LocalStack

LocalStack，顾名思义

> a local object with  a stack instead

也就是说，LocalStack和Local的工作方法差不多一样，Local以属性的形式存取线程资源，而LocalStack是以栈的形式存取线程资源。看看例子就明白了：

![local_and_localstack](/../img/local_and_localstack.png)

LocalStack源码如下，

```python
# part of werkzeug.local
class LocalStack(object):

    def __init__(self):
        self._local = Local()

    def push(self, obj):
        rv = getattr(self._local, 'stack', None)
        if rv is None:
            self._local.stack = rv = []
        rv.append(obj)
        return rv

    def pop(self):
        stack = getattr(self._local, 'stack', None)
        if stack is None:
            return None
        elif len(stack) == 1:
            release_local(self._local)
            return stack[-1]
        else:
            return stack.pop()

    @property
    def top(self):
        try:
            return self._local.stack[-1]
        except (AttributeError, IndexError):
            return None
    # ......
```

LocalStack的实现比较简单。

- 当我们创建一个LocalStack对象时，我们也创建了一个内部Local对象
- LocalStack对象push、pop、top等方法实际上也代理到内部Local对象上

### LocalProxy

LocalProxy，顾名思义，

> a localproxy acts as a proxy for a werkzeug local

关于代理，可以参考[这几篇文章](https://quanke.gitbooks.io/design-pattern-java/content/%E4%BB%A3%E7%90%86%E6%A8%A1%E5%BC%8F-Proxy%20Pattern.html)，它们很好地向我们阐述了代理模式。

一个LocalProxy对象就是一个Local对象的代理。我们对LocalProxy对象的操作都会转发到Local对象上。来看看例子吧：

![localproxy](/../img/localproxy.png)

上面，我们创建了一个Local对象`l`，给`l`添加个`request`属性；然后创建一个LocalProxy对象`lp`，代理到`l`的`request`上。我们对`lp`的操作都实际上作用到`l.request`上。

LocalProxy除了可以代理Local对象，还可以代理其它对象，只需将对象工厂函数传递给LocalProxy构造函数即可。见下：

![localproxy2](/../img/localproxy2.png)

那么，这个ProxyPython是怎么实现的呢？答案就在源码里：

```python
@implements_bool
class LocalProxy(object):
    __slots__ = ('__local', '__dict__', '__name__')

    def __init__(self, local, name=None):
        object.__setattr__(self, '_LocalProxy__local', local)
        object.__setattr__(self, '__name__', name)

    def _get_current_object(self):
        if not hasattr(self.__local, '__release_local__'):
            return self.__local()
        try:
            return getattr(self.__local, self.__name__)
        except AttributeError:
            raise RuntimeError('no object bound to %s' % self.__name__)

    def __getattr__(self, name):
        if name == '__members__':
            return dir(self._get_current_object())
        return getattr(self._get_current_object(), name)

    def __setitem__(self, key, value):
        self._get_current_object()[key] = value

    __setattr__ = lambda x, n, v: setattr(x._get_current_object(), n, v)
    __getitem__ = lambda x, i: x._get_current_object()[i]
    __iter__ = lambda x: iter(x._get_current_object())
    __contains__ = lambda x, i: i in x._get_current_object()
    __add__ = lambda x, o: x._get_current_object() + o
    __sub__ = lambda x, o: x._get_current_object() - o

    # ......

```

- 在构造函数`__init__`中，通过`__setattr__`函数，LocalProxy对象的`_LocalProxy__local`被设置为local，这个属性的名字比较奇怪，可以简单的认为为`self.__local = local`(具体可以参考[这里](http://docs.python.org/2/tutorial/classes.html#private-variables-and-class-local-references))
- LocalProxy对象通过`_get_current_object`获取代理的对象，在这个方法中，proxy通过判断代理对象是否含有`__release_local__`属性来判断代理对象是一个`werkzeug Local`对象还是一个普通对象。
- 接下来就是一大段一大段的重载魔术方法。LocalProxy几乎重载了所有的魔术方法，让所有关于它的操作都转发到通过`_get_current_object`获取的代理对象上。

### 小结

至此，我们通过阅读源码，以及提供简单的例子，学习了`werkzeug.local`中主要的类。先总结如下：

- `Local`：和标准库`threading.local`功能类似，但对greenlet也起作用
- `LocalStack`：和`Local`类似，但一般通过栈访问线程资源
- `LocalProxy`：一般涌来代理`Local`对象

The end.
