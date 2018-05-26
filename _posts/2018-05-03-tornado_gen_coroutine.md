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

{% highlight python linenos %}
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

{% highlight python linenos %}
"""tornado/gen.py"""
def sleep(duration):
    """ 返回一个在指定秒以后被解决的Future
    """
    # 创建一个Future
    f = _create_future()
    # 这里就是向IOLoop的定时器最小堆插入一个定时器任务:设置Future的结果为None
    IOLoop.current().call_later(duration,
                                lambda: future_set_result_unless_cancelled(f, None))
    return f
{% endhighlight %}

* 然后借助yield, 我们把控制权交给主协程, 在指定秒数后, Future被设置了结果, 然后控制权交回给本协程, 这些都是经由tornado.gen.coroutine这个装饰器来完成的

## tornado.gen.coroutine

* 在没有经由gen.coroutine装饰之前, my_sleep就是一个简单的生成器函数, 那么装饰之后是什么呢?

{% highlight python linenos %}
"""tornado/gen.py"""
def coroutine(func, replace_callback=True):
    """ 异步生成器装饰器
    """
    return _make_coroutine_wrapper(func, replace_callback=True)

def _make_coroutine_wrapper(func, replace_callback):
    """ coroutine装饰器的具体实现
    """
    # 用于兼容Python3.5 await
    wrapped = func
    if hasattr(types, 'coroutine'):
        func = types.coroutine(func)
    # 装饰之后的function
    @functools.wraps(wrapped)
    def wrapper(*args, **kwargs):
        # 创建一个Future, 用于存放被装饰的func的返回值
        future = _create_future()
        # 如果replace_callback为True且调用时的关键字参数中含有callback
        if replace_callback and 'callback' in kwargs:
            callback = kwargs.pop('callback')
            # 为Future添加回调函数
            IOLoop.current().add_future(
                future, lambda future: callback(future.result()))

        try:
            # 调用被装饰的function, 譬如之前的my_sleep生成器函数
            result = func(*args, **kwargs)
        except (Return, StopIteration) as e:
            # 如果抛出Return或StopIteration异常, 那么从异常中提取结果
            result = _value_from_stopiteration(e)
        except Exception:
            # 如果是其他异常, 设置Future的异常信息
            future_set_exc_info(future, sys.exc_info())
            # 返回Future, 以下写法如何避免循环引用？
            try:
                return future
            finally:
                future = None
        else:
            # 如果func是生成器函数, 那么调用后返回值必然是一个生成器
            if isinstance(result, GeneratorType):
                try:
                    # 获取原始的栈上下文
                    orig_stack_contexts = stack_context._state.contexts
                    # 从生成器中获取下一个item, 譬如上面my_sleep `yield tornado.gen.sleep(10)`返回的Future
                    yielded = next(result)
                    if stack_context._state.contexts is not orig_stack_contexts:
                        # 如果当前的栈上下文和next调用之前的栈上下文不同, 那么说明生成器改变了栈上下文,
                        # 导致了栈上下文的不一致?
                        yielded = _create_future()
                        yielded.set_exception(
                            stack_context.StackContextInconsistentError(
                                'stack_context inconsistency (probably caused '
                                'by yield within a "with StackContext" block)'))
                except (StopIteration, Return) as e:
                    # 如果抛出Return或StopIteration异常, 那么从异常中提取结果设置到Future
                    future_set_result_unless_cancelled(future, _value_from_stopiteration(e))
                except Exception:
                    # 如果是其他异常, 设置Future的异常信息
                    future_set_exc_info(future, sys.exc_info())
                else:
                    # 使用生成器、Future和本次生成的item创建一个协程运行器
                    runner = Runner(result, future, yielded)
                    # 为Future添加回调函数: (什么也不做), 用于为协程运行器添加一个强引用?
                    future.add_done_callback(lambda _: runner)
                yielded = None
                # 返回Future
                try:
                    return future
                finally:
                    future = None
        # 如果func是普通函数, 那么直接设置Future的结果为其返回值
        future_set_result_unless_cancelled(future, result)
        return future
    # 为装饰之后的function添加一些必要属性
    wrapper.__wrapped__ = wrapped
    wrapper.__tornado_coroutine__ = True
    # 返回装饰之后的function, 由此可见经由coroutine装饰后的生成器函数变成了一个返回Future的普通函数
    return wrapper
{% endhighlight %}

* 接下来我们再看看协程运行器到底是个什么东东:

{% highlight python linenos %}
"""tornado/gen.py"""

class Runner(object):
    """ 协程运行器, 就是对生成器进行了统一的封装
    """
    def __init__(self, gen, result_future, first_yielded):
        # gen就是生成器
        self.gen = gen
        # result_future是一个Future, 用于存放生成器的最终返回值
        self.result_future = result_future
        # future存放上一次yield的返回值
        self.future = _null_future
        self.yield_point = None
        self.pending_callbacks = None
        self.results = None
        self.running = False
        self.finished = False
        self.had_exception = False
        # 创建的时候, 记录当前的IOLoop
        self.io_loop = IOLoop.current()
        self.stack_context_deactivate = None
        # first_yielded就是生成器返回的第一个item, 这里使用handle_yield进行处理
        if self.handle_yield(first_yielded):
            gen = result_future = first_yielded = None
            self.run()

    def run(self):
        """ 开始运行或恢复被挂起的生成器, 使之一直运行, 直到某个yieldpoint未准备好
        """
        if self.running or self.finished:
            # 检查自身状态
            return
        try:
            # 设置running为True, 表明处于运行之中
            self.running = True
            while True:
                # 死循环
                future = self.future
                if not future.done():
                    # 如果上一次yield的返回值未准备好, 那么直接返回
                    return
                # 更新future为空
                self.future = None
                try:
                    orig_stack_contexts = stack_context._state.contexts
                    exc_info = None

                    try:
                        # 获取上一次yield的返回值
                        value = future.result()
                    except Exception:
                        self.had_exception = True
                        exc_info = sys.exc_info()
                    future = None

                    if exc_info is not None:
                        # 如果产生了异常, 那么通过throw方法向生成器抛出一个异常
                        try:
                            yielded = self.gen.throw(*exc_info)
                        finally:
                            exc_info = None
                    else:
                        # 如果未产生异常, 那么通过send方法向生成器返回value
                        yielded = self.gen.send(value)

                    if stack_context._state.contexts is not orig_stack_contexts:
                        # 检查栈上下文一致性
                        self.gen.throw(
                            stack_context.StackContextInconsistentError(
                                'stack_context inconsistency (probably caused '
                                'by yield within a "with StackContext" block)'))
                except (StopIteration, Return) as e:
                    # 如果生成器抛出StopIteration或Return异常, 那么表明生成器已执行完毕
                    # 置finished为True
                    self.finished = True
                    self.future = _null_future
                    if self.pending_callbacks and not self.had_exception:
                        # 如果还有等待执行的callback, 那么抛出异常
                        raise LeakedCallbackError(
                            "finished without waiting for callbacks %r" %
                            self.pending_callbacks)
                    # 从异常中提出value并设置为result_future的结果
                    future_set_result_unless_cancelled(self.result_future,
                                                       _value_from_stopiteration(e))
                    self.result_future = None
                    self._deactivate_stack_context()
                    return
                except Exception:
                    # 如果产生了其他异常
                    self.finished = True
                    self.future = _null_future
                    future_set_exc_info(self.result_future, sys.exc_info())
                    self.result_future = None
                    self._deactivate_stack_context()
                    return
                # 对当前生成器产生的yielded进行处理
                if not self.handle_yield(yielded):
                    # 如果未完成对yielded的处理, 那么退出循环
                    return
                yielded = None
        finally:
            self.running = False

    def handle_yield(self, yielded):
        """ 对生成器的返回值进行处理 """
        # 如果yielded是一个包含yieldpoint的容器, 也就是dict或者list
        if _contains_yieldpoint(yielded):
            yielded = multi(yielded)
        # YieldPoint就是生成器一类返回值的基类
        if isinstance(yielded, YieldPoint):
            # 创建一个新的Future
            self.future = Future()

            def start_yield_point():
                # 对YieldPoint进行处理
                try:
                    yielded.start(self)
                    if yielded.is_ready():
                        # 设置上一次yield的返回值为yielded的结果
                        future_set_result_unless_cancelled(self.future, yielded.get_result())
                    else:
                        # 更新yield_point
                        self.yield_point = yielded
                except Exception:
                    self.future = Future()
                    future_set_exc_info(self.future, sys.exc_info())

            if self.stack_context_deactivate is None:
                # 创建一个ExceptionStackContext实例作为栈上下文deactivate
                with stack_context.ExceptionStackContext(
                        self.handle_exception) as deactivate:
                    self.stack_context_deactivate = deactivate

                    def cb():
                        start_yield_point()
                        self.run()
                    # 为IOLoop添加一个callback, 并返回False, 因为并未对yieldpoint进行处, 而是放到了下一次IOLoop迭代中
                    self.io_loop.add_callback(cb)
                    return False
            else:
                # 对yielded进行处理
                start_yield_point()
        else:
            # 如果yielded不是YieldPoint, 譬如说Future
            try:
                # 将yielded转换为类Future对象
                self.future = convert_yielded(yielded)
            except BadYieldError:
                # 如果yielded无法进行转换
                self.future = Future()
                future_set_exc_info(self.future, sys.exc_info())

        if self.future is moment:
            # 如果future是一个特定对象moment, 那么将run放入IOLoop的callback队列
            self.io_loop.add_callback(self.run)
            return False
        elif not self.future.done():
            # 如果future未被处理, 那么为future添加回调函数: 将inner放入IOLoop的callback队列
            def inner(f):
                # Break a reference cycle to speed GC.
                f = None # noqa
                self.run()
            self.io_loop.add_future(
                self.future, inner)
            return False
        return True
{% endhighlight %}


* 从上面的分析我们可以得出装饰后的my_sleep被调用时的执行过程如下:

1. gen.sleep返回了一个Future, 并在IOLoop中添加了一个定时器任务: 设置Future的结果为None, 然后Future被yield出去, 也就是说生成器被挂起

2. 使用该Future创建了一个协程Runner, Runner对Future进行处理, 具体就是为Future添加一个回调函数:当Future完成时, 将Runner的run函数放入IOLoop的callback队列

3. 经过特定秒数后, 定时器任务被处理, Future被完成, 其回调函数将Runner的run函数放入IOLoop的callback队列

4. 在IOLoop的下一次迭代中, Runner的run函数作为callback被执行, 挂起的生成器被恢复运行, 如此反复, 直至生成器运行完毕