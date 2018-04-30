# Tornado Ioloop

Tornado推荐采用单进程单线程的运行方式; 为了充分利用CPU时间片, 使用了非阻塞IO, 而底层的Ioloop则基于IO多路复用模型


## 测试代码

    import tornado.ioloop

    def test():
        print("test start")
        print("test end")
    
    # 创建(获取)当前线程的Ioloop实例
    loop = tornado.ioloop.IOLoop.current()
    loop.add_callback(test)
    loop.start()


## Ioloop创建过程

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
        “”“
        获取当前线程的Ioloop实例,
        如果当前线程尚未创建Ioloop实例且参数instance为True, 那么就创建一个实例
        “”“
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
            “”“
            使当前Ioloop实例处于current位置！
            原则上来说, 在同一个进程其实是可以创建多个Ioloop实例的, 但是线程中处
            于运行状态的只能有一个; 通过让Ioloop实例处于current位置, 我们可以方便
            的将各种callback、timeout和fd注册到当前线程的同一个Ioloop实例上！
            有点宣誓主权的意思
            
            “”“
            # The asyncio event loops override this method.
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
    “”“基于轮询的IOLoop“”“
    
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


## Ioloop注册回调函数

    import collections
    
    class PollIOLoop(IOLoop):
    “”“基于轮询的IOLoop“”“
    
        def initialize(self, impl, time_func=None, **kwargs):
            “”“PollIOLoop实例的初始化方法”“”
            
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
            “”“注册回调函数“”“
            
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

## IoLoop主循环

    class PollIOLoop(IOLoop):
    “”“基于轮询的IOLoop“”“
    
        def initialize(self, impl, time_func=None, **kwargs):
            “”“PollIOLoop实例的初始化方法”“”
            
            super(PollIOLoop, self).initialize(**kwargs)
            self._impl = impl
            if hasattr(self._impl, 'fileno'):
                set_close_exec(self._impl.fileno())
            self.time_func = time_func or time.time
            self._handlers = {}
            self._events = {}
            self._callbacks = collections.deque()
            self._timeouts = []
            self._cancellations = 0
            self._running = False
            self._stopped = False
            self._closing = False
            self._thread_ident = None
            self._pid = os.getpid()
            self._blocking_signal_threshold = None
            self._timeout_counter = itertools.count()

            # _waker为管道, 用于向运行IoLoop的处于阻塞状态的线程发送消息, 以唤醒它
            self._waker = Waker()
            self.add_handler(self._waker.fileno(),
                             lambda fd, events: self._waker.consume(),
                             self.READ)
    
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
                # requires python 2.6+, unix.  set_wakeup_fd exists but crashes
                # the python process on windows.
                try:
                    old_wakeup_fd = signal.set_wakeup_fd(self._waker.write_fileno())
                    if old_wakeup_fd != -1:
                        # Already set, restore previous value.  This is a little racy,
                        # but there's no clean get_wakeup_fd and in real use the
                        # IOLoop is just started once at the beginning.
                        signal.set_wakeup_fd(old_wakeup_fd)
                        old_wakeup_fd = None
                except ValueError:
                    # Non-main thread, or the previous value of wakeup_fd
                    # is no longer valid.
                    old_wakeup_fd = None

            try:
                while True:
                    # Prevent IO event starvation by delaying new callbacks
                    # to the next iteration of the event loop.
                    ncallbacks = len(self._callbacks)

                    # Add any timeouts that have come due to the callback list.
                    # Do not run anything until we have determined which ones
                    # are ready, so timeouts that call add_timeout cannot
                    # schedule anything in this iteration.
                    due_timeouts = []
                    if self._timeouts:
                        now = self.time()
                        while self._timeouts:
                            if self._timeouts[0].callback is None:
                                # The timeout was cancelled.  Note that the
                                # cancellation check is repeated below for timeouts
                                # that are cancelled by another timeout or callback.
                                heapq.heappop(self._timeouts)
                                self._cancellations -= 1
                            elif self._timeouts[0].deadline <= now:
                                due_timeouts.append(heapq.heappop(self._timeouts))
                            else:
                                break
                        if (self._cancellations > 512 and
                                self._cancellations > (len(self._timeouts) >> 1)):
                            # Clean up the timeout queue when it gets large and it's
                            # more than half cancellations.
                            self._cancellations = 0
                            self._timeouts = [x for x in self._timeouts
                                              if x.callback is not None]
                            heapq.heapify(self._timeouts)

                    for i in range(ncallbacks):
                        self._run_callback(self._callbacks.popleft())
                    for timeout in due_timeouts:
                        if timeout.callback is not None:
                            self._run_callback(timeout.callback)
                    # Closures may be holding on to a lot of memory, so allow
                    # them to be freed before we go into our poll wait.
                    due_timeouts = timeout = None

                    if self._callbacks:
                        # If any callbacks or timeouts called add_callback,
                        # we don't want to wait in poll() before we run them.
                        poll_timeout = 0.0
                    elif self._timeouts:
                        # If there are any timeouts, schedule the first one.
                        # Use self.time() instead of 'now' to account for time
                        # spent running callbacks.
                        poll_timeout = self._timeouts[0].deadline - self.time()
                        poll_timeout = max(0, min(poll_timeout, _POLL_TIMEOUT))
                    else:
                        # No timeouts and no callbacks, so use the default.
                        poll_timeout = _POLL_TIMEOUT

                    if not self._running:
                        break

                    if self._blocking_signal_threshold is not None:
                        # clear alarm so it doesn't fire while poll is waiting for
                        # events.
                        signal.setitimer(signal.ITIMER_REAL, 0, 0)

                    try:
                        event_pairs = self._impl.poll(poll_timeout)
                    except Exception as e:
                        # Depending on python version and IOLoop implementation,
                        # different exception types may be thrown and there are
                        # two ways EINTR might be signaled:
                        # * e.errno == errno.EINTR
                        # * e.args is like (errno.EINTR, 'Interrupted system call')
                        if errno_from_exception(e) == errno.EINTR:
                            continue
                        else:
                            raise

                    if self._blocking_signal_threshold is not None:
                        signal.setitimer(signal.ITIMER_REAL,
                                         self._blocking_signal_threshold, 0)

                    # Pop one fd at a time from the set of pending fds and run
                    # its handler. Since that handler may perform actions on
                    # other file descriptors, there may be reentrant calls to
                    # this IOLoop that modify self._events
                    self._events.update(event_pairs)
                    while self._events:
                        fd, events = self._events.popitem()
                        try:
                            fd_obj, handler_func = self._handlers[fd]
                            handler_func(fd_obj, events)
                        except (OSError, IOError) as e:
                            if errno_from_exception(e) == errno.EPIPE:
                                # Happens when the client closes the connection
                                pass
                            else:
                                self.handle_callback_exception(self._handlers.get(fd))
                        except Exception:
                            self.handle_callback_exception(self._handlers.get(fd))
                    fd_obj = handler_func = None

            finally:
                # reset the stopped flag so another start/stop pair can be issued
                self._stopped = False
                if self._blocking_signal_threshold is not None:
                    signal.setitimer(signal.ITIMER_REAL, 0, 0)
                if old_current is None:
                    IOLoop.clear_current()
                elif old_current is not self:
                    old_current.make_current()
                if old_wakeup_fd is not None:
                    signal.set_wakeup_fd(old_wakeup_fd)