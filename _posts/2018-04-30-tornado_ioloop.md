---
layout: post
title: Tornado Ioloop学习
date: 2018-04-30
category: "Tornado"
tags: [Web Server,Tornado]
author: Lambda
comment: false
---

# Tornado Ioloop

Tornado推荐采用单进程单线程的运行方式; 为了充分利用CPU时间片, 使用了非阻塞IO, 而底层的Ioloop则基于IO多路复用模型


## 测试代码

```
import tornado.ioloop

def test():
    print("test start")
    print("test end")

# 创建(获取)当前线程的Ioloop实例
loop = tornado.ioloop.IOLoop.current()
loop.add_callback(test)
loop.start()
```


## Ioloop创建过程

```
"""tornado/ioloop.py"""
import threading

# asyncio是Python 3.4版本引入的标准库, 直接内置了对异步IO的支持;
# 尝试导入asyncio
try:
    import asyncio
except ImportError:
    asyncio = None

class IOLoop(Configurable):
    # 线程相关数据(TSD)
    _current = threading.local()
    
    _ioloop_for_asyncio = dict()
    
    @staticmethod
    def current(instance=True):
    """
    获取当前线程的Ioloop实例,
    如果当前线程尚未创建Ioloop实例且参数instance为True, 那么就创建一个实例
    """
        if asyncio is None:
            # 如果asyncio为None, 说明当前版本Python尚不支持异步IO
            
            # 从_current线程相关数据中获取instance属性
            current = getattr(IOLoop._current, "instance", None)
            if current is None and instance:
                # 如果current为None, 说明当前线程尚未创建Ioloop实例;
                # 并且参数instance为True, 进行创建
                
                # IOLoop的实例到底是个什么东西呢?
                # 1.可以认真学习下tornado/util.py模块的Configurable类, 该类
                # 一般作为基类存在, 旨在通过继承来创建可配置的派生类
                # 2.通过configurable_default函数我们可以看出, Ioloop会优先使用
                # 基于asyncio的AsyncIOLoop, 在不支持asyncio的环境下才会使用
                # PollIOLoop
                # 3.在通过PollIOLoop类的configurable_default函数, 我们可以看出
                # 其会优先使用基于epoll的EPollIOLoop, 其次是基于kqueue的KQueueIOLoop,
                # 最后则是基于select的SelectIOLoop
                # 4.综上所述, 其实IOLoop的实例具体是什么, 是由当前Python的运行时
                # 决定的;
                
                # 创建Ioloop实例后, 还会调用make_current方法来使当前实例被置于
                # current位置, 那么接下来通过current方法获取的实例就是当前创建
                # 的实例
                current = IOLoop()
                if IOLoop._current.instance is not current:
                    # 如果当前实例没有被置于current位置, 那么说明发生了运行时错误
                    raise RuntimeError("new IOLoop did not become current")
        else:
            # 当前运行时支持asyncio, 那么使用asyncio内置的ioloop;
            # 由于对asyncio不太熟悉, 所以这部分内容后续补充
            try:
                loop = asyncio.get_event_loop()
            except (RuntimeError, AssertionError):
                if not instance:
                    return None
                raise
            try:
                return IOLoop._ioloop_for_asyncio[loop]
            except KeyError:
                if instance:
                    from tornado.platform.asyncio import AsyncIOMainLoop
                    current = AsyncIOMainLoop(make_current=True)
                else:
                    current = None
        # 返回处于current位置的Ioloop实例
        return current
        
    def make_current(self):
        """
        使当前Ioloop实例处于current位置！
        原则上来说, 在同一个进程其实是可以创建多个Ioloop实例的, 但是线程中处
        于运行状态的只能有一个; 通过让Ioloop实例处于current位置, 我们可以方便
        的将各种callback、timeout和fd注册到当前线程的同一个Ioloop实例上！
        有点宣誓主权的意思
        """
        assert asyncio is None
        old = getattr(IOLoop._current, "instance", None)
        if old is not None:
            old.clear_current()
        IOLoop._current.instance = self
    
    
    @classmethod
    def configurable_default(cls):
        if asyncio is not None:
            from tornado.platform.asyncio import AsyncIOLoop
            return AsyncIOLoop
        return PollIOLoop
        

class PollIOLoop(IOLoop):
"""基于轮询的IOLoop"""

    @classmethod
    def configurable_default(cls):
        if hasattr(select, "epoll"):
            # 支持epoll的情况下使用EPollIOLoop;
            # Linux 2.6版本引入了epoll
            from tornado.platform.epoll import EPollIOLoop
            return EPollIOLoop
        if hasattr(select, "kqueue"):
            # 支持kqueue的情况下使用KQueueIOLoop;
            # FreeBSD才支持kqueue
            from tornado.platform.kqueue import KQueueIOLoop
            return KQueueIOLoop
        # 最后才使用基于select的SelectIOLoop;
        # select在大部分系统中都得到了支持, 但是性能会弱于以上两者
        from tornado.platform.select import SelectIOLoop
        return SelectIOLoop
```


## Ioloop注册回调函数

```
import collections

class PollIOLoop(IOLoop):
"""基于轮询的IOLoop"""

    def initialize(self, impl, time_func=None, **kwargs):
        """PollIOLoop实例的初始化方法"""
        
        ...
        
        # 创建一个双头队列_callbacks来存放回调函数
        self._callbacks = collections.deque()

        self._closing = False
        self._thread_ident = None

        ...
        
        self._waker = Waker()
        self.add_handler(self._waker.fileno(),
                         lambda fd, events: self._waker.consume(),
                         self.READ)

    def add_callback(self, callback, *args, **kwargs):
        """注册回调函数"""
        
        if self._closing:
            # 如果_closing为True, 表明当前IoLoop实例已经被关闭了
            return

        # 将callback封装成partial函数, 然后追加到_callbacks队列中,
        # deque的append方法是原子性的, 因此是线程安全的
        self._callbacks.append(functools.partial(
            stack_context.wrap(callback), *args, **kwargs))
        if thread.get_ident() != self._thread_ident:
            # 如果当前线程和运行此IoLoop的线程不是同一个, 那么我们通过_waker来
            # 唤醒运行此IoLoop的线程！
            # 运行IoLoop的线程可能阻塞在poll上, 而callback的优先级相对较高,
            # 那么我们需要通过signal去唤醒它, 使之能及时对callback进行处理
            self._waker.wake()
        else:
            # 如果当前线程就是运行此IoLoop的线程, 那么就没有必要进行唤醒操作了
            pass
```


## IoLoop主循环

```
class PollIOLoop(IOLoop):
"""基于轮询的IOLoop"""

    def initialize(self, impl, time_func=None, **kwargs):
        """PollIOLoop实例的初始化方法"""
        
        super(PollIOLoop, self).initialize(**kwargs)
        self._impl = impl
        if hasattr(self._impl, 'fileno'):
            set_close_exec(self._impl.fileno())
        self.time_func = time_func or time.time
        self._handlers = {}
        self._events = {}
        self._callbacks = collections.deque()
        # 使用最小堆维护的定时器任务
        self._timeouts = []
        self._cancellations = 0
        self._running = False
        self._stopped = False
        self._closing = False
        self._thread_ident = None
        self._pid = os.getpid()
        self._blocking_signal_threshold = None
        self._timeout_counter = itertools.count()

        # _waker顾名思义为唤醒器, 用于向运行IoLoop的处于阻塞状态的线程发送消息, 以唤醒它;
        # nt系统下是通过一对C/S socket实现的; posix系统则是通过pipe实现的
        self._waker = Waker()
        self.add_handler(self._waker.fileno(),
                         lambda fd, events: self._waker.consume(),
                         self.READ)
    
    def _run_callback(self, callback):
        """对回调进行处理"""
        
        try:
            ret = callback()
            if ret is not None:
                from tornado import gen
                try:
                    ret = gen.convert_yielded(ret)
                except gen.BadYieldError:
                    pass
                else:
                    self.add_future(ret, self._discard_future_result)
        except Exception:
            self.handle_callback_exception(callback)

    def start(self):
        """运行IoLoop的主循环"""
        if self._running:
            # 如果_running为True, 那么意味着IoLoop实例正在运行之中;
            # 我们不允许对处于运行状态的IoLoop实例多次执行start方法
            raise RuntimeError("IOLoop is already running")
        if os.getpid() != self._pid:
            # 如果调用start方法的进程和运行IoLoop实例的进程不是同一个,
            # 那么报运行时错误: PollIOLoop实例不能被多个进程共享
            raise RuntimeError("Cannot share PollIOLoops across processes")
        # 设置日志
        self._setup_logging()
        if self._stopped:
            # 如果_stopped为True, 说明当前IoLoop实例之前运行过然后被停止了,
            # 可能担心受前一次运行的影响, 这里直接就返回了
            self._stopped = False
            return
        # 在运行之前, 需要确保当前IoLoop实例处于current位置
        old_current = IOLoop.current(instance=False)
        if old_current is not self:
            self.make_current()
        # 设置_thread_ident为当前线程的ident
        self._thread_ident = thread.get_ident()
        # 设置_running为True, 表明自己处于运行状态
        self._running = True


        old_wakeup_fd = None
        if hasattr(signal, 'set_wakeup_fd') and os.name == 'posix':
            # 只有在POSIX操作系统上时, 才会调用set_wakeup_fd接口, Windows系
            # 统上会导致Python进程崩溃?
            try:
                # set_wakeup_fd用于唤醒select或poll, 设置一个非阻塞的fd, 每
                # 当有信号到来时, 往该fd写入'\0', 返回值为先前设置的fd
                old_wakeup_fd = signal.set_wakeup_fd(self._waker.write_fileno())
                if old_wakeup_fd != -1:
                    # 如果old_wakeup_fd不为-1, 表明之前已经通过set_wakeup_fd
                    # 设置过fd, 进一步表明IoLoop可能已经开始, 所以进行恢复！
                    # 主要是没有get_wakeup_fd类似的接口, 否则不会采取这样的
                    # 实现形式
                    signal.set_wakeup_fd(old_wakeup_fd)
                    old_wakeup_fd = None
            except ValueError:
                # 非主线程或者先前设置的wakeup_fd已失效
                old_wakeup_fd = None

        try:
            # IoLoop的死循环, 以下内容和大部分基于事件的网络库/服务程序中
            # 的死循环大同小异, 譬如Nginx、eventlet等
            while True:
                # 这里用于记录此轮迭代要处理的回调个数,
                # 之后在处理回调和定时器任务时追加的回调会延迟到下一轮迭代
                # 中处理, 防止I/O事件被饿死
                ncallbacks = len(self._callbacks)

                # due_timeouts用于存放已超时的定时器任务
                due_timeouts = []
                if self._timeouts:
                    # 如果_timeouts不为空, 即表明有已经注册的定时器任务
                    
                    # 获取当前时间
                    now = self.time()
                    while self._timeouts:
                        # 定时器任务deadline越小优先级越高, 需要优先处理
                        # 这里就是按照deadline从小到大的顺序遍历_timeouts
                    
                        if self._timeouts[0].callback is None:
                            # 如果定时器任务的callback为空, 即表明该定时器
                            # 任务已被取消, 那么从最小堆中移除该定时器
                            # 注:由于是最小堆, 那么_timeouts的第一个元素即为
                            # 最小值, 所以heappop移除的正是该最小值
                            heapq.heappop(self._timeouts)
                            # _cancellations记录的是_timeouts中已取消的定时
                            # 器任务个数, 移除后就需要减1
                            self._cancellations -= 1
                        elif self._timeouts[0].deadline <= now:
                            # 表明定时器任务已超时, 那么从_timeouts中移除并
                            # 追加到due_timeouts中
                            due_timeouts.append(heapq.heappop(self._timeouts))
                        else:
                            # 如果当前定时器任务没有超时, 那么_timeouts中剩
                            # 余的定时器任务肯定不会超时了, 退出遍历
                            break
                    if (self._cancellations > 512 and
                            self._cancellations > (len(self._timeouts) >> 1)):
                        # Clean up the timeout queue when it gets large and it's
                        # more than half cancellations.
                        # 如果_timeouts中剩余的已取消的定时器任务个数超过512
                        # 且超过_timeouts长度的一半, 那么说明_timeouts中有大
                        # 量的无用元素, 这些无用元素会对最小堆的调整产生负面
                        # 影响, 从而对IoLoop的性能造成影响, 因此需要移除这些
                        # 无用元素, 并重新构造最小堆
                        self._cancellations = 0
                        self._timeouts = [x for x in self._timeouts
                                          if x.callback is not None]
                        heapq.heapify(self._timeouts)
                
                # 对前ncallbacks个回调进行处理
                for i in range(ncallbacks):
                    self._run_callback(self._callbacks.popleft())
                # 对已经超时的定时器任务进行处理
                for timeout in due_timeouts:
                    if timeout.callback is not None:
                        self._run_callback(timeout.callback)
                # 释放无用的内存
                due_timeouts = timeout = None

                # 确定poll的超时时间
                if self._callbacks:
                    # 如果回调队列不为空, 说明有很重要的事情需要处理,
                    # 那么poll的超时时间为0, 即使用非阻塞的poll
                    poll_timeout = 0.0
                elif self._timeouts:
                    # 如果注册的定时器任务不为空, 那么为了避免定时器任务无
                    # 法被及时处理, 设置poll的超时时间为定时器任务的最小deadline
                    # 距离当前的时间, 另外这个时间不能超过_POLL_TIMEOUT
                    poll_timeout = self._timeouts[0].deadline - self.time()
                    poll_timeout = max(0, min(poll_timeout, _POLL_TIMEOUT))
                else:
                    # 如果回调队列为空且没有注册的定时器任务, 那么采用默认
                    # 的poll超时时间_POLL_TIMEOUT
                    poll_timeout = _POLL_TIMEOUT

                if not self._running:
                    # 如果_running为False, 即表明IoLoop被停止
                    break

                if self._blocking_signal_threshold is not None:
                    # _blocking_signal_threshold不为空, 即表明在poll期间清空
                    # alarm以防止内核发送SIGALRM信号
                    signal.setitimer(signal.ITIMER_REAL, 0, 0)

                try:
                    # _impl可能为epoll、kqueue或者select等实现,
                    # 这里进行轮询获取已产生的事件
                    event_pairs = self._impl.poll(poll_timeout)
                except Exception as e:
                    # 如果异常为EINTR, 那么表明poll使用的系统调用被信号中断,
                    # 这样的异常是正常的; 其他异常则抛出
                    if errno_from_exception(e) == errno.EINTR:
                        continue
                    else:
                        raise

                if self._blocking_signal_threshold is not None:
                    # 恢复alarm
                    signal.setitimer(signal.ITIMER_REAL,
                                     self._blocking_signal_threshold, 0)
                
                # 使用_events来记录产生的事件?
                # 猜测是: 兼容使用边沿触发的情况, 避免事件丢失
                self._events.update(event_pairs)
                # 对产生的事件进行遍历和处理
                while self._events:
                    fd, events = self._events.popitem()
                    try:
                        # _handlers作为字典, 记录了注册的fd->fd_obj,handler_func
                        # 的映射关系, handler_func就是事件处理函数, fd_obj
                        # 就是fd对应的file-like对象, 譬如socket对象; fd只是
                        # 个整型数据, 通常我们的事件处理函数不会面向它
                        fd_obj, handler_func = self._handlers[fd]
                        # 使用注册的事件处理函数对fd_obj上产生的events事件
                        # 进行处理
                        handler_func(fd_obj, events)
                    except (OSError, IOError) as e:
                        if errno_from_exception(e) == errno.EPIPE:
                            # 当客户端关闭连接时
                            pass
                        else:
                            # 对事件处理函数产生的异常进行处理
                            self.handle_callback_exception(self._handlers.get(fd))
                    except Exception:
                        self.handle_callback_exception(self._handlers.get(fd))
                # 设置fd_obj和handler_func为None, 使其原来引用的对象的ref_count
                # 减1, 促使其能尽早被GC回收以释放占用的内存空间
                fd_obj = handler_func = None

        finally:
            # 退出死循环后
            
            # 置_stopped为False, 那么另一对start/stop操作可以发起
            self._stopped = False
            if self._blocking_signal_threshold is not None:
                # 清空alarm
                signal.setitimer(signal.ITIMER_REAL, 0, 0)
                
            if old_current is None:
                # 如果old_current为None, 那么直接clear_current即可
                IOLoop.clear_current()
            elif old_current is not self:
                # 如果old_current不为空且不是自己, 那么说明在运行之前有另一
                # 个IoLoop实例已经占据了current位置, 在运行之后需要进行恢复
                old_current.make_current()
            if old_wakeup_fd is not None:
                # old_wakeup_fd不为None时只能为-1, 这里表示清空wakeup_fd
                signal.set_wakeup_fd(old_wakeup_fd)
```