---
layout:     post
title:      【译】async/await 到底是怎么工作的呢？
subtitle:   How the heck does async/await work in Python 3.5?
author:     JX
pub_date: 2017-08-23
tags:
    - async/await
---

> 之前对于 async/await 总是一知半解，看了很多博客文章也不清楚其原理。偶遇[How the heck does async/await work in Python 3.5?](https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/)，如醍醐灌顶，豁然开朗。故翻译此文，以供学习探讨。
>
>
>
> 原文：[How the heck does async/await work in Python 3.5?](https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/)

我知道 [`yield from` in Python 3.3](https://docs.python.org/3/whatsnew/3.3.html#pep-380)和[`asyncio` in Python 3.4](https://docs.python.org/3/library/asyncio.html#module-asyncio) 联合在一起衍生出了 `async`/`await` 这个新语法。例如，我知道：

```python
yield from iterator
```

本质上等于：

```python
for x in iterator:
    yield x
```

我也知道`asyncio`是一个允许异步编程的事件循环框架。我知道它们各自的含义，但是当它们组装在一起衍生出`async/await`语法时，我就懵了。于是，我决定花些时间来研究 [`async`/`await` ](https://docs.python.org/3/whatsnew/3.5.html#whatsnew-pep-492)到底是怎么工作的。现在，因为我想要更好的理解`async/await`是怎么工作的，所以这篇文章会设计到一些底层的CPython技术细节。



## Python中协程 coroutines 的历史

根据 [Wikipedia](https://www.wikipedia.org/)解释：

> [Coroutines](https://en.wikipedia.org/wiki/Coroutine) are computer program components that generalize subroutines for nonpreemptive multitasking, by allowing multiple entry points for suspending and resuming execution at certain locations.

简单说，一个协程就是一个函数，但是你可以决定什么时候终止函数的执行，什么时候恢复该函数的执行。这时，你可能会说，“这听起来很像生成器generators啊”。嗯，你是对的。



回到 [Python 2.2](https://docs.python.org/3/whatsnew/2.2.html)，生成器generators是在 [PEP 255](https://www.python.org/dev/peps/pep-0255/) 中引进来的（那时生成器叫做 generator iterators，因为它们实现了迭代器协议[iterator protocol](https://docs.python.org/3/library/stdtypes.html#iterator-types)）。受到[Icon语言](http://www.cs.arizona.edu/icon/)的启发，生成器可以通过一个超级简单的方式实现一个迭代器，这个迭代器不会占用很多的内存资源，但可以很快地获取下一个值（你也可以通过写一个实现了`__iter__()`和`__next()`两个魔术方法的类来实现一个迭代器，不过这样做相对来说比较费力）。

在引入生成器以前，如果你要实现一个自己的`range()`函数，你可以简单地创建一个整数列表就能实现了：

```python
def eager_range(up_to):
    """创建一个从 0 到 up_to 的整数列表，不包括 up_to"""
    sequence = []
    index = 0
    while index < up_to:
        sequence.append(index)
        index += 1
    return sequence
```

但是，这样做的问题是，如果你想要一个很大的序列，比如从0到1000000之间的整数，你就需要创建一个足够大的列表来容纳这么多数，这是非常占用内存资源的。当生成器被引入到Python后，你所需要的就是一个整数大小的内存：

```python
def lazy_range(up_to):
    """返回一个从0到up_to(不包含)序列的生成器"""
    index = 0
    while index < up_to:
        yield index
        index += 1
```

当函数遇到`yield`表达式时，它就停下来了，以后又能从停止的地方继续执行，就内存使用而言，这时非常有用的，因为它只用到一个整数的内存大小。

如你所见，目前我们见到的生成器都是关于迭代器的。在 [PEP 342](https://www.python.org/dev/peps/pep-0342/)中 ，传送一些值到一个停止的生成器这个特性终于添加到[Python 2.5](https://docs.python.org/3/whatsnew/2.5.html)。这时候，Python就有了协程这个概念了。 PEP 342 在生成器中引入了 `send()` 方法。 这样，你不仅可以暂停一个生成器，还可以在它暂停的地方给它传送一些数据并继续执行。

还是以`range()`为例，你可以向生成器发送一个数据从而向前活着向后进行跳转：

```python
def jumping_range(up_to):
    """0～up_to（不包含）的生成器，向生成器发送一个数据进行跳转"""
    index = 0
    while index < up_to:
        jump = yield index
        if jump is None:
            jump = 1
        index += jump

if __name__ == '__main__':
    iterator = jumping_range(5)
    print(next(iterator))  # 0
    print(iterator.send(2))  # 2
    print(next(iterator))  # 3
    print(iterator.send(-1))  # 2
    for x in iterator:
        print(x)  # 3, 4
```

到了 [Python 3.3](https://docs.python.org/3/whatsnew/3.3.html) ， [PEP 380](https://www.python.org/dev/peps/pep-0380/) 添加了`yield from`语法，生成器的功能就更加丰富了。如果你的生成器要从一个迭代器产生每一个值，`yield from`这个特性能够让你更简洁明了地重构生成器。

```python
def lazy_range(up_to):
    """0～up_to(不包含)的生成器"""
    index = 0
    def gratuitous_refactor():
        nonlocal index
        while index < up_to:
            yield index
            index += 1
    yield from gratuitous_refactor()
```

`yield from` 也能够让你将多个生成器连接起来。

```python
def bottom():
    return (yield 42)

def middle():
    return (yield from bottom())

def top():
    return (yield from middle())

# 获取生成器
gen = top()
value = next(gen)
print(value)  # Prints '42'.
try:
    value = gen.send(value * 2)
except StopIteration as exc:
    value = exc.value
print(value)  # Prints '84'.
```

### 小结

- Python 2.2 中引进来的生成器让代码的执行能够被中止。
- Python 2.5 中，我们可以向一个中止的生成器发送值，这时候有了协程这个概念。
- Python 3.3 引进了`yield from`，我们可以更方便的重构和链接生成器。



## 什么是事件循环？

理解事件循环是什么以及它是怎样让异步编程成为可能，这对理解`async`/`await`是非常重要的。如果你之前有过图形界面GUI编程，包括web前端编程，那么你已经就是在使用事件循环进行编程了。

Wikipedia是这么定义事件循环的：

> an [event loop](https://en.wikipedia.org/wiki/Event_loop) "is a programming construct that waits for and dispatches events or messages in a program".

简单说，事件循环就是“当事件A发生时，执行事件B”。我们现在就以浏览器的 JavaScript 事件循环为例：

1. 当你点击某个东西时，这个点击事件`click`会被JavaScript事件循环收集（当事件A发生）
2. JavaScript会去检查是否有任何回调函数`onclick`注册在该点击事件上，有就调用该回调函数（执行事件B）

在Python中，标准库`asyncio` 提供了一个事件循环。`asyncio`事件循环主要用于处理网络请求，这里的“当事件A发生”指的是：当来自于套接字的I/O处于可读和／或者可写状态（使用 [`selectors`](https://docs.python.org/3/library/selectors.html#module-selectors)模块）。除了GUIs和IO，事件循环还可以用于执行其它线程或者子进程的代码，也就是说把事件循环当作一个调度器 （例如， [cooperative multitasking](https://en.wikipedia.org/wiki/Cooperative_multitasking)）。

### 小结

- 事件循环就是当事件循环关心的事件发生时，事件循环会去调用其他关心这个事件的回调函数。
- Python 3.4 以标准库`asyncio`的形式获得了一个事件循环。



##  `async/await` 是怎样工作的

## Python 3.4 怎样实现`async/await`相同的效果呢

由于Python 3.3 提供的生成器特性以及`asyncio`提供的事件循环， Python 3.4 有足够的能力来支持（以并发编程的形式 [concurrent programming](https://en.wikipedia.org/wiki/Concurrent_computing)）异步编程。异步编程是说代码块的运行顺序我们事先是不知道的，因此称为异步而不是同步。 并发编程*Concurrent programming* 是编写代码，让它们之间可以独立运行而不依赖与其他部分，尽管这些代码是在同一个线程中运行的 ([concurrency is **not** parallelism](http://blog.golang.org/concurrency-is-not-parallelism))。例如，下边就是Python 3.4异步编程的例子：在两个异步并发的函数调用中实现一个减数器。

```python
import asyncio

# 代码来自于 http://curio.readthedocs.org/en/latest/tutorial.html.
@asyncio.coroutine
def countdown(number, n):
    while n > 0:
        print('T-minus', n, '({})'.format(number))
        yield from asyncio.sleep(1)
        n -= 1

loop = asyncio.get_event_loop()
tasks = [
    asyncio.ensure_future(countdown("A", 2)),
    asyncio.ensure_future(countdown("B", 3))]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```

在 Python 3.4 中， 装饰器 [`asyncio.coroutine`](https://docs.python.org/3/library/asyncio-task.html#asyncio.coroutine) 就是用来把一个函数标记为协程 [coroutine](https://docs.python.org/3/reference/datamodel.html?#coroutine-objects) ，这个协程将会在 `asyncio` 以及它提供的事件循环中使用。这也让 Python 有了一个明确具体的关于协程的定义：an object who implemented the methods added to generators in [PEP 342](https://www.python.org/dev/peps/pep-0342/) and represented by the [`collections.abc.Coroutine` abstract base class](https://docs.python.org/3/library/collections.abc.html#collections.abc.Coroutine). This meant that suddenly all generators implemented the coroutine interface even if they weren't meant to be used in that fashion. To fix this, `asyncio` required that all generators meant to be used as a coroutine had to be [decorated with `asyncio.coroutine`](https://docs.python.org/3/library/asyncio-task.html#asyncio.coroutine).

有了这个明确的协程定义（这个定义和生成器的API接口很匹配），你可以在任意 [`asyncio.Future`](https://docs.python.org/3/library/asyncio-task.html#future)对象（asyncio中会具体定义future对象，我们不用花太多心思理解它）上使用`yield from`来将该对象传递给`asyncio`事件循环，并中止协程，然后等待新的调用。一旦 future 对象到达事件循环，这个future对象将会被监控，直到它的状态变为`已完成`；一旦future对象变成`已完成`状态，事件循环将会使用`send()`函数将future对象的返回值传递给之前中止的协程，让该协程重新执行。

重新思考我们上面的例子：

- 事件循环开始执行两个 `countdown()` 协程，直到执行到其中一个协程的 `yield from` 表达式和 `asyncio.sleep()` 函数。这会返回一个`asyncio.Future` 对象给事件循环并中止该协程的运行。
- 事件循环将会监控该future对象，直到一秒后。一秒后，事件循环把该future对象的返回值发送到上一步中止的协程，并从中止的地方重新执行该协程。
- 上面者两个步骤将会一直持续循环下去，直到两个 `countdown()` 协程运行完毕，并且该事件循环没有其他需要监控的future对象。

我将会向你展示一个完整的关于协程和事件循环的例子，但我还是想先阐述下 `async` 的 `await` 工作原理。

### Python 3.4 怎样实现`async/await`相同的效果呢

由于Python 3.3 提供的生成器特性以及`asyncio`提供的事件循环， Python 3.4 有足够的能力来支持（以并发编程的形式 [concurrent programming](https://en.wikipedia.org/wiki/Concurrent_computing)）异步编程。异步编程是说代码块的运行顺序我们事先是不知道的，因此称为异步而不是同步。 并发编程*Concurrent programming* 是编写代码，让它们之间可以独立运行而不依赖与其他部分，尽管这些代码是在同一个线程中运行的 ([concurrency is **not** parallelism](http://blog.golang.org/concurrency-is-not-parallelism))。例如，下边就是Python 3.4异步编程的例子：在两个异步并发的函数调用中实现一个减数器。

```python
import asyncio

# 代码来自于 http://curio.readthedocs.org/en/latest/tutorial.html.
@asyncio.coroutine
def countdown(number, n):
    while n > 0:
        print('T-minus', n, '({})'.format(number))
        yield from asyncio.sleep(1)
        n -= 1

loop = asyncio.get_event_loop()
tasks = [
    asyncio.ensure_future(countdown("A", 2)),
    asyncio.ensure_future(countdown("B", 3))]
loop.run_until_complete(asyncio.wait(tasks))
loop.close()
```

在 Python 3.4 中， 装饰器 [`asyncio.coroutine`](https://docs.python.org/3/library/asyncio-task.html#asyncio.coroutine) 就是用来把一个函数标记为协程 [coroutine](https://docs.python.org/3/reference/datamodel.html?#coroutine-objects) ，这个协程将会在 `asyncio` 以及它提供的事件循环中使用。这也让 Python 有了一个明确具体的关于协程的定义：an object who implemented the methods added to generators in [PEP 342](https://www.python.org/dev/peps/pep-0342/) and represented by the [`collections.abc.Coroutine` abstract base class](https://docs.python.org/3/library/collections.abc.html#collections.abc.Coroutine). This meant that suddenly all generators implemented the coroutine interface even if they weren't meant to be used in that fashion. To fix this, `asyncio` required that all generators meant to be used as a coroutine had to be [decorated with `asyncio.coroutine`](https://docs.python.org/3/library/asyncio-task.html#asyncio.coroutine).

有了这个明确的协程定义（这个定义和生成器的API接口很匹配），你可以在任意 [`asyncio.Future`](https://docs.python.org/3/library/asyncio-task.html#future)对象（asyncio中会具体定义future对象，我们不用花太多心思理解它）上使用`yield from`来将该对象传递给`asyncio`事件循环，并中止协程，然后等待新的调用。一旦 future 对象到达事件循环，这个future对象将会被监控，直到它的状态变为`已完成`；一旦future对象变成`已完成`状态，事件循环将会使用`send()`函数将future对象的返回值传递给之前中止的协程，让该协程重新执行。

重新思考我们上面的例子：

- 事件循环开始执行两个 `countdown()` 协程，直到执行到其中一个协程的 `yield from` 表达式和 `asyncio.sleep()` 函数。这会返回一个`asyncio.Future` 对象给事件循环并中止该协程的运行。
- 事件循环将会监控该future对象，直到一秒后。一秒后，事件循环把该future对象的返回值发送到上一步中止的协程，并从中止的地方重新执行该协程。
- 上面者两个步骤将会一直持续循环下去，直到两个 `countdown()` 协程运行完毕，并且该事件循环没有其他需要监控的future对象。

我将会向你展示一个完整的关于协程和事件循环的例子，但我还是想先阐述下 `async` 的 `await` 工作原理

### Python 3.5 把`yield from` 替换成 `await`

在Python 3.4中，如果我们想要异步编程，我们可以给一个函数加上一个装饰器`asyncio.coroutine`让它成为一个协程：

```python
# 这在 Python 3.5 也生效
@asyncio.coroutine
def py34_coro():
    yield from stuff()
```

在Python 3.5中，装饰器 [types.coroutine](https://docs.python.org/3/library/types.html#types.coroutine)也可以把一个函数标识为协程，就跟`asyncio.coroutine` 一样。此外，我们还可以使用`async def`来标识函数为协程，不过这个函数不能包含任何`yield`或者`yield from`表达式，只可以用`return`或者`await`来返回值。

```python
async def py35_coro():
    await stuff()
```

 关于`async` 和 `types.coroutine`最重要的一件事是，它更加准确和具体地定义了协程（ tighten the definition）。  It takes coroutines from simply being an interface to an actual type, making the distinction between any generator and a generator that is meant to be a coroutine much more stringent (and the [`inspect.iscoroutine()` function](https://docs.python.org/3/library/inspect.html#inspect.iscoroutine) is even stricter by saying `async` has to be used).

在上面这个例子中，除了 `async` ，你应该也注意到了Python 3.5引进了 `await`表达式（ `await`只有在一个`async def`才生效）。

While `await` operates much like `yield from`, the objects that are acceptable to an `await` expression are different. Coroutines are definitely allowed in an `await` expression since the concept of coroutines are fundamental in all of this. But when you call `await` on an object , it technically needs to be an [*awaitable* object](https://docs.python.org/3/reference/datamodel.html?#awaitable-objects): an object that defines an `__await__()` method which returns an iterator which is **not** a coroutine itself . Coroutines themselves are also considered awaitable objects (hence why `collections.abc.Coroutine` inherits from `collections.abc.Awaitable`). This definition follows a Python tradition of making most syntax constructs translate into a method call underneath the hood, much like `a + b` is `a.__add__(b)` or `b.__radd__(a)` underneath it all.

 `yield from` 和 `await` 之间的区别，让我们从更底层的角度来分析吧。从字节编码的角度重新审查上面两个协程例子，探索其本质区别。

 `py34_coro()`的字节编码是:

```assembly
>>> dis.dis(py34_coro)
  2           0 LOAD_GLOBAL              0 (stuff)
              3 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
              6 GET_YIELD_FROM_ITER
              7 LOAD_CONST               0 (None)
             10 YIELD_FROM
             11 POP_TOP
             12 LOAD_CONST               0 (None)
             15 RETURN_VALUE
```

 `py35_coro()` 的字节编码为 :

```assembly
>>> dis.dis(py35_coro)
  1           0 LOAD_GLOBAL              0 (stuff)
              3 CALL_FUNCTION            0 (0 positional, 0 keyword pair)
              6 GET_AWAITABLE
              7 LOAD_CONST               0 (None)
             10 YIELD_FROM
             11 POP_TOP
             12 LOAD_CONST               0 (None)
             15 RETURN_VALUE
```

忽略由于 `py34_coro()` 的`asyncio.coroutine` 装饰器引起的行号不同，唯一的不同是：

- [`GET_YIELD_FROM_ITER` opcode](https://docs.python.org/3/library/dis.html#opcode-GET_YIELD_FROM_ITER)
-  [`GET_AWAITABLE` opcode](https://docs.python.org/3/library/dis.html#opcode-GET_AWAITABLE).

 Both functions are properly flagged as being coroutines, so there's no difference there. In the case of `GET_YIELD_FROM_ITER`, it simply checks if its argument is a generator or coroutine, otherwise it calls `iter()` on its argument (the acceptance of a coroutine object by the opcode for `yield from` is only allowed when the opcode is used from within a coroutine itself, which is true in this case thanks to the `types.coroutine` decorator flagging the generator as such at the C level with the `CO_ITERABLE_COROUTINE` flag on the code object).

But `GET_AWAITABLE` does something different. While the bytecode will accept a coroutine just like `GET_YIELD_FROM_ITER`, it will **not** accept a generator if has not been flagged as a coroutine. Beyond just coroutines, though, the bytecode will accepted an *awaitable* object as discussed earlier. This makes `yield from` expressions and `await` expressions both accept coroutines while differing on whether they accept plain generators or awaitable objects, respectively.

You may be wondering why the difference between what an `async`-based coroutine and a generator-based coroutine will accept in their respective pausing expressions? The key reason for this is to make sure you don't mess up and accidentally mix and match objects that just happen to have the same API to the best of Python's abilities. Since generators inherently implement the API for coroutines then it would be easy to accidentally use a generator when you actually expected to be using a coroutine. And since not all generators are written to be used in a coroutine-based control flow, you need to avoid accidentally using a generator incorrectly. But since Python is not statically compiled, the best the language can offer is runtime checks when using a generator-defined coroutine. This means that when `types.coroutine` is used, Python's compiler can't tell if a generator is going to be used as a coroutine or just a plain generator (remember, just because the syntax says `types.coroutine` that doesn't mean someone hasn't earlier done `types = spam` earlier), and thus different opcodes that have different restrictions are emitted by the compiler based on the knowledge it has at the time.

One very key point I want to make about the difference between a generator-based coroutine and an `async` one is that only generator-based coroutines can actually pause execution and force something to be sent down to the event loop. You typically don't see this very important detail because you usually call event loop-specific functions like the [`asyncio.sleep()` function](https://docs.python.org/3/library/asyncio-task.html#asyncio.sleep) since event loops implement their own APIs and these are the kind of functions that have to worry about this little detail. For the vast majority of us, we will work with event loops rather than be writing them and thus only be writing `async` coroutines and never need to really care about this. But if you're like me and were wondering why you couldn't write something like `asyncio.sleep()` using only `async` coroutines, this can be quite the "aha!" moment.

### 小结

使用`async def`定义一个函数可以使它成为一个协程。另一个方式是使用 `types.coroutine`（底层是在代码对象上使用`CO_ITERABLE_COROUTINE`）或者 `collections.abc.Coroutine`的子类来装饰一个生成器。

可等待`awaitable`对象要么是一个协程，要么是一个实现了 `__await__()`的对象。可等待对象返回的是一个迭代器，而不是一个协程。

`await`表达式和`yield from`表达式作用相似，但它只在可等待对象上生效（普通的生成器在 `await` 表达式中不起效）。

 `async` 函数就是一个协程，它里边要么包含一个 `return` 语句（包括隐式的return None），要么包含一个 `await` 表达式 ，两者都包含也可以（也就是说，`yield` 表达式是不被允许的）。这么做的原因是为了防止我们混用两种不同形式的生成器：

- 普通生成器
- 基于生成器的协程

因为这两种形式的生成器的作用是大不相同的。


## 把 `async`/`await` 当作异步编程API接口

直到看了[David Beazley's Python Brasil 2015 keynote](https://www.youtube.com/watch?v=lYe8W04ERnY)这个演讲，我才深入思考这个问题。在这个演讲中，David指出，`async`/`await` 本质上就是一个异步编程接口。David 想表达的是，我们不应该把 `async`/`await` 当作 `asyncio`的同义词，而要这么认为， `asyncio`是一个利用了 `async`/`await` 接口的异步编程框架。

David actually believes this idea of `async`/`await` being an asynchronous programming API that he has created the [`curio` project](https://pypi.python.org/pypi/curio) to implement his own event loop. This has helped make it clear to me that `async`/`await` allows Python to provide the building blocks for asynchronous programming, but without tying you to a specific event loop or other low-level details (unlike other programming languages which integrate the event loop into the language directly).

这就允许 `curio` 这样的项目能够以一种不同的底层实现方式来实现（例如， `asyncio` 使用了future对象作为和事件循环通信的接口，而 `curio` 使用tuples）， but to also have different focuses and performance characteristics (e.g., `asyncio` has an entire framework for implementing transport and protocol layers which makes it extensible while `curio` is simpler and expects the user to worry about that kind of thing but also allows it to run faster).

Based on the (short) history of asynchronous programming in Python, it's understandable that people might think that `async`/`await` == `asyncio`. I mean `asyncio` was what helped make asynchronous programming possible in Python 3.4 and was a motivating factor for adding `async`/`await` in Python 3.5. But the design of `async`/`await` is purposefully flexible enough to **not** require `asyncio` or contort any critical design decision just for that framework. In other words, `async`/`await` continues Python's tradition of designing things to be as flexible as possible while still being pragmatic to use (and implement).

## 例子

At this point your head might be awash with new terms and concepts, making it a little hard to fully grasp how all of this is supposed to work to provide you asynchronous programming. To help make it all much
more concrete, here is a complete (if contrived) asynchronous programming example, end-to-end from event loop and associated functions to user code. The example has coroutines which represents individual
rocket launch countdowns but that appear to be counting down simultaneously . This is asynchronous programming through concurrency; three separate coroutines will be running independently, and yet it will
 all be done in a single thread.

```python
import datetime
import heapq
import types
import time


class Task:

    """Represent how long a coroutine should wait before starting again.

    Comparison operators are implemented for use by heapq. Two-item
    tuples unfortunately don't work because when the datetime.datetime
    instances are equal, comparison falls to the coroutine and they don't
    implement comparison methods, triggering an exception.

    Think of this as being like asyncio.Task/curio.Task.
    """

    def __init__(self, wait_until, coro):
        self.coro = coro
        self.waiting_until = wait_until

    def __eq__(self, other):
        return self.waiting_until == other.waiting_until

    def __lt__(self, other):
        return self.waiting_until < other.waiting_until


class SleepingLoop:

    """An event loop focused on delaying execution of coroutines.

    Think of this as being like asyncio.BaseEventLoop/curio.Kernel.
    """

    def __init__(self, *coros):
        self._new = coros
        self._waiting = []

    def run_until_complete(self):
        # Start all the coroutines.
        for coro in self._new:
            wait_for = coro.send(None)
            heapq.heappush(self._waiting, Task(wait_for, coro))
        # Keep running until there is no more work to do.
        while self._waiting:
            now = datetime.datetime.now()
            # Get the coroutine with the soonest resumption time.
            task = heapq.heappop(self._waiting)
            if now < task.waiting_until:
                # We're ahead of schedule; wait until it's time to resume.
                delta = task.waiting_until - now
                time.sleep(delta.total_seconds())
                now = datetime.datetime.now()
            try:
                # It's time to resume the coroutine.
                wait_until = task.coro.send(now)
                heapq.heappush(self._waiting, Task(wait_until, task.coro))
            except StopIteration:
                # The coroutine is done.
                pass


@types.coroutine
def sleep(seconds):
    """Pause a coroutine for the specified number of seconds.

    Think of this as being like asyncio.sleep()/curio.sleep().
    """
    now = datetime.datetime.now()
    wait_until = now + datetime.timedelta(seconds=seconds)
    # Make all coroutines on the call stack pause; the need to use `yield`
    # necessitates this be generator-based and not an async-based coroutine.
    actual = yield wait_until
    # Resume the execution stack, sending back how long we actually waited.
    return actual - now


async def countdown(label, length, *, delay=0):
    """Countdown a launch for `length` seconds, waiting `delay` seconds.

    This is what a user would typically write.
    """
    print(label, 'waiting', delay, 'seconds before starting countdown')
    delta = await sleep(delay)
    print(label, 'starting after waiting', delta)
    while length:
        print(label, 'T-minus', length)
        waited = await sleep(1)
        length -= 1
    print(label, 'lift-off!')


def main():
    """Start the event loop, counting down 3 separate launches.

    This is what a user would typically write.
    """
    loop = SleepingLoop(countdown('A', 5), countdown('B', 3, delay=2),
                        countdown('C', 4, delay=1))
    start = datetime.datetime.now()
    loop.run_until_complete()
    print('Total elapsed time is', datetime.datetime.now() - start)


if __name__ == '__main__':
    main()
```

As I said, it's contrived, but if you run this in Python 3.5 you will notice that all three coroutines run independently in a single thread and yet the total amount of time taken to run is about 5 seconds. You
can consider `Task`, `SleepingLoop`, and `sleep()` as what an event loop provider like `asyncio` and `curio` would give you. For a normal user, only the code in `countdown()` and `main()` are of importance. As you can see, there is no magic to `async`, `await`, or this whole asynchronous programming deal; it's just an API that Python provides you to help make this sort of thing easier.

## 我的希望

Now that I understand how this asynchronous programming works in Python, I want to use it all the time! It's such an awesome concept that's so much better than something you would have used threads for previously. The problem is that Python 3.5 is so new that `async`/`await` is also very new. That means there are not a lot of libraries out there supporting asynchronous programming like this. For instance, to do HTTP requests you either have to construct the HTTP request yourself by hand (yuck), use a project like the [`aiohttp` framework](https://pypi.python.org/pypi/aiohttp) which adds HTTP on top of another event loop (in this case, `asyncio`), or hope more projects like the [`hyper` library](https://pypi.python.org/pypi/hyper) continue to spring up to provide an abstraction for things like HTTP which allow you to use whatever I/O library you want (although unfortunately `hyper` only supports HTTP/2 at the moment).

Personally, I hope projects like `hyper` take off so that we have a clear separation between getting binary data from I/O and how we interpret that binary data. This kind of abstraction is important because most I/O libraries in Python are rather tightly coupled to how they do I/O and how they handle data coming from I/O. This is a problem with the [`http` package in Python's standard library](https://docs.python.org/3/library/http.html#module-http) as it doesn't have an HTTP parser but a connection object which does all the I/O for you. And if you were hoping `requests` would support asynchronous programming, [your hopes have already been dashed](https://github.com/kennethreitz/requests/issues/2801) because the synchronous I/O that `requests` uses is baked into its design. This shift in ability to do asynchronous programming gives the Python community a chance to fix a problem it has with not having abstractions at the various layers of the network stack. And we have the perk of it not being hard to make asynchronous code run as if its synchronous, so tools filling the void for asynchronous programming can work in both worlds.

I also hope that Python gains some form of support in `async` coroutines for `yield`. Maybe this will require yet another keyword (maybe something like `anticipate`?), but the fact that you actually can't implement an event loop system with just `async` coroutines bothers me. Luckily, it turns out [I'm not the only one who thinks this](https://twitter.com/dabeaz/status/696014754557464576), and since the author of [PEP 492](https://www.python.org/dev/peps/pep-0492/) agrees with me, I think there's a chance of getting this quirk removed.

## 总结

Basically `async` and `await` are fancy generators that we call coroutines and there is some extra support for things called awaitable objects and turning plain generators in to coroutines. All of this comes together to support concurrency so that we have better support for asynchronous programming in Python. It's awesome and much easier to use than comparable approaches like threads -- I wrote an end-to-end example of asynchronous programming in under 100 lines of commented Python code -- while still being quite flexible and fast (the [curio FAQ](http://curio.readthedocs.org/en/latest/#questions-and-answers) says that it runs faster than `twisted` by 30-40% but slower than `gevent` by 10-15%, and all while being implemented in pure Python; remember that [Python 2 + Twisted can use less memory and is easier to debug than Go](https://news.ycombinator.com/item?id=10402307), so just imagine what you could do with this!). I'm very happy that this landed in Python 3 and I look forward to the community embracing it and helping to flesh out its support in libraries and frameworks so we can
all benefit from asynchronous programming in Python.


