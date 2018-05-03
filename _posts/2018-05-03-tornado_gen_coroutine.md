---
layout: post
title: 揭开Tornado gen.coroutine的神秘面纱
date: 2018-05-03
category: "Tornado"
tags: [Web Server,Tornado]
author: Lambda
comment: false
---

# Tornado gen.coroutine

tornado.gen中提供了基于generator(生成器)的协程实现, 使得我们可以按照同步代码的写法实现异步的效果, 而不用通过回调函数来实现.

由于tornado不像eventlet/gevent使用了greenlet并对原生Python库进行了patch, 所以tornado的同步写法并不像它们那样彻底, 以至于我们需要了解哪些代码可能阻塞tornado主线程、代码在哪些地方应该执行yield操作.


## 测试代码

{% highlight python %}
import tornado.gen
import tornado.ioloop

@tornado.gen.coroutine
def my_sleep():
    print("my_sleep start")
    # sleep 10 seconds
    yield tornado.gen.sleep(10)
    print("my_sleep end")

def hello():
    print("hello world")

# 创建(获取)当前线程的Ioloop实例
loop = tornado.ioloop.IOLoop.current()
loop.add_callback(my_sleep)
loop.add_callback(hello)
loop.start()
{% endhighlight %}


## 代码输出

    my_sleep start
    hello world
    my_sleep end


## 我的疑惑

* tornado.gen.sleep(10)用于让子程序暂停10秒, 如果换成time.sleep(10)会怎么样, 究竟是怎么做到的呢?

* my_sleep中带有yield也就是说它是一个生成器函数, 在经过tornado.gen.coroutine装饰后, my_sleep会变成什么呢?

## tornado.gen.sleep

* 由于tornado是单进程单线程的, 如果我们使用标准库time.sleep(10), 那么就会阻塞tornado主线程10秒, 这段时间tornado什么也做不了; 

* 如果我们的初衷仅仅是让my_sleep中两句print间隔10秒执行, 那么time.sleep(10)显然是不合适的, tornado.gen.sleep(10)才是正解.

* 让我们看看tornado.gen.sleep做了什么:

{% highlight python %}
"""tornado/gen.py"""
def sleep(duration):
    """Return a `.Future` that resolves after the given number of seconds.
    When used with ``yield`` in a coroutine, this is a non-blocking
    analogue to `time.sleep` (which should not be used in coroutines
    because it is blocking)::

        yield gen.sleep(0.5)

    Note that calling this function on its own does nothing; you must
    wait on the `.Future` it returns (usually by yielding it).

    .. versionadded:: 4.1
    """
    f = _create_future()
    IOLoop.current().call_later(duration,
                                lambda: future_set_result_unless_cancelled(f, None))
    return f
{% endhighlight %}
