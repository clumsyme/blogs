事件循环(event loop)是 `asyncio` 模块的核心。事件循环会运行异步任务和回调，执行网络 IO 操作，以及运行子进程。

在这篇文章内，我们暂时先不关注 `协程`，而是了解一下什么是事件循环、从一个用户的角度看事件循环可以做哪些工作、事件循环的实现原理等。

## 什么是事件循环

> In computer science, the event loop is a programming construct or design pattern that waits for and dispatches events or messages in a program. 
> ———— Event loop - Wikipedia

简单来说，事件循环用于等待并分发事件/消息的程序，所以有时候也被称作*消息分发器*。它向事件提供方发送请求并等待有事件发生时来调用相应的事件处理器。

从用户的角度看，事件循环就是不停的调用回调函数。

```
callbacks1 ---> callbacks2 ---> ... ---> callbacksN
```

## 用户角度的 `asyncio` 事件循环

为了实际了解 `asyncio` 中的事件循环怎么实现类似上边的 `callbacks` 调用的，我们来创建一个事件循环并调用一些 API 来使用它。

### 创建一个 event loop

```py
import asyncio

# 会在当前线程创建一个 event loop
loop = asyncio.get_event_loop()
```

### 启动 event loop

```py
loop.run_forever()
```

恰如方法的字面意思，event loop 会永远运行下去。它会阻塞当前程序。但是它什么也没有做，因为我们并没有在它上边注册任何事件。

### 让 event loop 运行一些东西

```py
loop.run_until_complete(asyncio.sleep(5))
```

事件循环会 等待 sleep ，直到 5 秒钟后 sleep 完成，然后它结束运行。
所以你可以让事件循环一直运行下去，也可以给它一些任务告诉它：运行这个任务直到它完成。

### 添加一些回调函数

```py
import datetime

def print_now():
    print(datetime.datetime.now())

loop.call_soon(print_now)
loop.call_later(1, print_now)
loop.run_until_complete(asyncio.sleep(5))
```

我们调用 `loop.call_soon` 告诉事件循环：尽快运行我给你的这个callback。

或者调用 `loop.call_later` 告诉事件循环：在指定时间后运行 这个 callback。

这就是我们如何注册 callback 给 event loop 的。

运行上述代码，会输出：

```
2020-04-27 15:08:57.903450
2020-04-27 15:08:58.904478

```

然后 Python 会阻塞 4 秒钟，直到 sleep 完成。

### 递归注册

```py
def trampoline(name = ''):
    print(name, end=' ')
    print_now()
    loop.call_later(1, trampoline, name)

loop.call_soon(trampoline)
# 3 秒钟后我们停止 event loop
loop.call_later(3, loop.stop)
loop.run_forever()

# 输出：
#  2020-04-27 15:16:12.261536
#  2020-04-27 15:16:13.263419
#  2020-04-27 15:16:14.264162
```

我们只注册了 `trampoline` 函数一次，但是它却运行了 3 次，因为在函数内部，它自己又注册了自己。

### 顺序

使用 call_soon 注册的 callback 调用顺序是先进先出，所以如果我们注册三个 `trampoline` 函数，可以发现在每次调用的时候他们的顺序都保持一致。

```py
# 之前 trampoline 函数在上一个 loop 上还有注册的 callback，即使 loop 被 stop 了，它再次运行的时候还是会被调用，影响我们接下来的输出。所以我们新建一个 event loop 来继续演示
loop = asyncio.new_event_loop()     

loop.call_soon(trampoline, 'First')
loop.call_soon(trampoline, 'Second')
loop.call_soon(trampoline, 'Third')
loop.call_later(3, loop.stop)

loop.run_forever()

# 输出
# First 2020-04-27 15:25:35.776239
# Second 2020-04-27 15:25:35.776293
# Third 2020-04-27 15:25:35.776314
# First 2020-04-27 15:25:36.779673
# Second 2020-04-27 15:25:36.779790
# Third 2020-04-27 15:25:36.779834
# First 2020-04-27 15:25:37.780068
# Second 2020-04-27 15:25:37.780173
# Third 2020-04-27 15:25:37.780214
# First 2020-04-27 15:25:38.781099
# Second 2020-04-27 15:25:38.781219
# Third 2020-04-27 15:25:38.781265
```

### 阻塞

事件循环在同一时间只会处理一个回调函数，所以如果某一个回调函数运行时间太长，它就会阻塞当前事件循环。

```py
loop = asyncio.new_event_loop()     

def hog():
    [i*j for i in range(10000) for j in range(10000)]
loop.call_soon(trampoline)    
loop.call_later(3, hog)
loop.call_later(5, loop.stop)
loop.run_forever()

# 输出
#  2020-04-27 15:33:22.669346
#  2020-04-27 15:33:23.671479
#  2020-04-27 15:33:24.673388
#  2020-04-27 15:33:34.727684
```

注意上边第三个和第四个输出，整整隔了 10 秒而非 1 秒。我们应该避免在事件循环中运行长时间的任务。

这就是用户角度的事件循环：事件循环接受一些注册的 callback，在合适的时机（尽快或者一定时间后）调用这些 callback；callback 按照注册的顺序进行调用；可以在 callback 内注册 callback；事件循环可以一直运行，直到我们调用它的`stop`方法，或者一直运行直到所有 callback 完成。事件循环在一个时间只能处理一个 callback，如果一个 callback 占用 CPU 较多，事件循环会被阻塞。

## `asyncio` 事件循环实现

记得上一篇讲述事件循环是基于 I/O 多路复用的[`select` 系统调用](https://www.imliyan.com/blogs/article/%E7%90%86%E8%A7%A3%20Python%20%E7%9A%84%20asyncio%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E4%B8%BA%E4%BB%80%E4%B9%88%E9%9C%80%E8%A6%81%20asyncio/#select)吧？在不同的平台下 `selector.select` 使用的是不同的实现，每一个事件循环实例都有一个 `self._selector` 属性，对应的就是该事件循环所使用的 `selector`。所以在不同平台上就有不同的事件循环实现。

```
                                AbstractEventLoop(定义事件循环接口)
                                            |
                                            |
                                    BaseEventLoop(基本事件循环)
                                            |
                                            |
                       |—————————————————————————————————————————|
                       |                                         |
                       |                                         |
            BaseSelectorEventLoop                   BaseProactorEventLoop
                       |                                         |
                       |                                         |
            ——————————————————————————                           |
           |                          |                          |
           |                          |                          |
           |                          |                          |
_UnixSelectorEventLoop      _WindowsSelectorEventLoop      ProactorEventLoop

```

在 macOS 上，`asyncio.get_event_loop()` 会返回一个 `_UnixSelectorEventLoop`，其 `_selector` 属性是一个 `selector.DefaultSelector()`，而它实际上是一个 `KqueueSelector`。
在 Windows 上，我们也可以使用 `_WindowsSelectorEventLoop` 但是它只支持套接字。所以 Windows 上的通常是使用由 IOCP 实现的 `ProactorEventLoop`，其 `_selector` 属性是一个 `IocpProactor`。

<!-- 我们大可不必关心各个平台的不同实现差别，因为这些差异 event loop -->

那么我们现在就看一下 CPython 源码里边 EventLoop 是怎么实现的吧。

- EventLoop

在 macOS 上，我们这样获取一个 event loop。


```py
loop = asyncio.get_event_loop()

print(loop)
# <_UnixSelectorEventLoop running=False closed=False debug=False>
```

- `run_forever`

`_UnixSelectorEventLoop` 并没有实现 `run_forever` 方法，而是直接继承自 `BaseSelectorEventLoop`，而 `BaseSelectorEventLoop` 则继承自 `BaseEventLoop`。

```py
class BaseEventLoop(events.AbstractEventLoop):
    def __init__(self):
        ...
        # 存放 ready 状态 callbacks 的队列
        self._ready = collections.deque()
        # 存放通过 call_later 等方法注册的 callbacks 的优先队列
        self._scheduled = []
        ...
    
    def run_forever(self):
        """Run until stop() is called."""
        ...

        try:
            ...
            while True:
                self._run_once()
                if self._stopping:
                    break
        finally:
            ...
```

关键代码就是一个 `while True` 的无限循环，直到它被 `stop`。

- `_run_once`

我们再看 `_run_once`方法：

根据该方法的 doc string，我们知道每一次调用 `_call_once`，会有四件事情发生：

- 使用 selector（或者proactor）来 select I/O 事件
- 处理这些事件（实际上是添加的_ready 队列）
- 调度 call_later 里的 callbacks
- 调用所以当前已经 ready 的 callbacks

```py
class BaseEventLoop(events.AbstractEventLoop):
    def _run_once(self):
        """Run one full iteration of the event loop.

        This calls all currently ready callbacks, polls for I/O,
        schedules the resulting callbacks, and finally schedules
        'call_later' callbacks.
        """
        ...

        timeout = None
        if self._ready or self._stopping:
            timeout = 0
        elif self._scheduled:
            # Compute the desired timeout.
            when = self._scheduled[0]._when
            timeout = min(max(0, when - self.time()), MAXIMUM_SELECT_TIMEOUT)

        # 这一行代码就是 asyncio 核心的核心，使用 selector 来 select I/O 事件
        event_list = self._selector.select(timeout)

        self._process_events(event_list)

        # Handle 'later' callbacks that are ready.
        end_time = self.time() + self._clock_resolution
        while self._scheduled:
            handle = self._scheduled[0]
            if handle._when >= end_time:
                break
            handle = heapq.heappop(self._scheduled)
            handle._scheduled = False
            self._ready.append(handle)

        # This is the only place where callbacks are actually *called*.
        # All other places just add them to ready.
        # Note: We run all currently scheduled callbacks, but not any
        # callbacks scheduled by callbacks run this time around --
        # they will be run the next time (after another I/O poll).
        # Use an idiom that is thread-safe without using locks.
        ntodo = len(self._ready)
        for i in range(ntodo):
            handle = self._ready.popleft()
            ...
            handle._run()
```

1. 使用 `selector`（或者proactor）来 `select` I/O 事件

`self._selector.select(timeout)` 的 timeout 参数，取决于 `self._ready`、`self._stopping` 和 `self._scheduled`。

如果 `self._ready` 不为空，说明已经有准备好的 callbacks 等待我们去执行了，或者 loop 是 stop 状态，那么 timeout 为 0，由上一篇文章可知，timeout 小于等于 0 时，`select` 方法会立即返回 ready 状态的 I/O 事件，不会在此阻塞。

如果 loop 没有 stop，`self._ready` 也为空，但是 `self._scheduled` 不为空，说明我们有在将来准备调用的 callbacks，但是现在还没有准备好的。那么 timeout 为 `self._scheduled` 里距离现在时间最短的 callback 的相对现在的时间（最小为 0，最大是 24 小时）。这时候，`select`函数会阻塞 timeout 时间，等待下一个 callback 变为 ready，或者 selector 监听到了新的 I/O 事件。

如果上边两个条件都不满足，既没有准备好的，也没有注册的将来调用的 callbacks，timeout 会是默认值 None，意味着 `select` 会无限期阻塞，一直到监听到了新的 I/O 事件。

在 `select` 得到 `event_list`之后，Python 会 `_process_events`，实际上就是调用 `self._add_callback` 将这些 event 放到 事件循环的 `_ready` 队列里。

2. 处理这些事件： `_process_events` & `self._add_callback`

我们来看一下 `_process_events`:

```py
class BaseSelectorEventLoop(base_events.BaseEventLoop):
    def _process_events(self, event_list):
        for key, mask in event_list:
            fileobj, (reader, writer) = key.fileobj, key.data
            # 准备好读
            if mask & selectors.EVENT_READ and reader is not None:
                if reader._cancelled:
                    self._remove_reader(fileobj)
                else:
                    self._add_callback(reader)
            # 准备好写
            if mask & selectors.EVENT_WRITE and writer is not None:
                if writer._cancelled:
                    self._remove_writer(fileobj)
                else:
                    self._add_callback(writer)

class BaseEventLoop(events.AbstractEventLoop):
    def _add_callback(self, handle):
        """Add a Handle to _scheduled (TimerHandle) or _ready."""
        ...

        self._ready.append(handle)
```

3. 处理 call_later 里已经准备好的事件

会把 `self._scheduled` 里已经准备好的添加到 `self._ready` 队列。

```py
# Handle 'later' callbacks that are ready.
end_time = self.time() + self._clock_resolution
while self._scheduled:
    handle = self._scheduled[0]
    if handle._when >= end_time:
        break
    handle = heapq.heappop(self._scheduled)
    handle._scheduled = False
    self._ready.append(handle)
```

4. 调用准备好的 callbacks

这里才是真正、并且唯一的 callback 被调用的地方。之前的不管是 `call_soon` 或者 `call_later`，都只是把 callback 放到队列里，而不是去执行。

```py
ntodo = len(self._ready)
for i in range(ntodo):
    handle = self._ready.popleft()
    ...
    handle._run()
```

比如下边测试：

```py
def say_hi():
    print('Hi')

def test():
    print('start')
    loop.call_soon(say_hi)
    print('end')

loop = asyncio.get_event_loop()
loop.call_soon(test)
loop.run_forever()

# 输出：
# start
# end
# Hi
```

### 我们来看下 `call_soon` 和 `call_later` 的具体实现

- `call_soon` 会使用 Handle 来包装 callback，并且把包装后的 handle 对象添加到 `self._ready` 队列。

```py
class BaseEventLoop(events.AbstractEventLoop):
    def call_soon(self, callback, *args, context=None):
        """Arrange for a callback to be called as soon as possible.

        This operates as a FIFO queue: callbacks are called in the
        order in which they are registered.  Each callback will be
        called exactly once.

        Any positional arguments after the callback will be passed to
        the callback when it is called.
        """
        ...
        handle = self._call_soon(callback, args, context)
        ...
        return handle
    def _call_soon(self, callback, args, context):
        # 使用 Handle 包装 callback
        handle = events.Handle(callback, args, self, context)
        ...
        # 将 callback 放到 _ready 队列里
        self._ready.append(handle)
        return handle
```

- `call_later` 会实际调用 `call_at`，将相对时间 `delay` 转变为绝对时间。然后使用 `TimerHandle` 来包装 callback，然后就包装后的 timer 对象放到 `self._scheduled` 优先队列里。

```py
class BaseEventLoop(events.AbstractEventLoop):
   def call_later(self, delay, callback, *args, context=None):
        """Arrange for a callback to be called at a given time.

        Return a Handle: an opaque object with a cancel() method that
        can be used to cancel the call.

        The delay can be an int or float, expressed in seconds.  It is
        always relative to the current time.

        Each callback will be called exactly once.  If two callbacks
        are scheduled for exactly the same time, it undefined which
        will be called first.

        Any positional arguments after the callback will be passed to
        the callback when it is called.
        """
        timer = self.call_at(self.time() + delay, callback, *args,
                             context=context)
        ...
        return timer

    def call_at(self, when, callback, *args, context=None):
        """Like call_later(), but uses an absolute time.

        Absolute time corresponds to the event loop's time() method.
        """
        ...
        # 使用 TimerHandle 包装 callback
        timer = events.TimerHandle(when, callback, args, self, context)
        ...
        # 将 timer 放入 self._scheduled 优先队列里
        heapq.heappush(self._scheduled, timer)
        timer._scheduled = True
        return timer
```

所以这两个函数的实现也说明 `call_soon` 和 `call_later` 仅仅是将 callback 包装后放入队列。在每一次循环调用 `_run_once` 的时候才处理 `self._ready` 里的 callbacks。

### *关于 `_schduled`*

`_schduled` 里边保存的是会在将来执行的 callback，它是一个优先队列结构，里边的对象是包装类 callback 的 TimerHandle。它在优先队列里排序的依据是 `when` 属性，也就是一个绝对时间。所以这个优先队列在 pop 的时候会按照时间先后顺序依次 pop。

```py
class TimerHandle(Handle):
    def __lt__(self, other):
        if isinstance(other, TimerHandle):
            return self._when < other._when
        return NotImplemented
```

## 总结：

一个事件循环有一个 `_ready` 双端队列保存当前准备好执行的 callbacks，还有一个 `_scheduled` 优先队列保存注册了准备在将来执行的 callbacks，还有一个 `_selector` 在调用 `select` 的时候会得到当前准备好的 I/O 事件。

事件循环 `run_forever` 的时候，会循环调用 `_run_once` 方法，而每次 `_run_once` 都会做四件事：

- 使用 `select` 得到准备好的 I/O 事件
- 将获取到的事件放到 `_ready` 队列里
- 检查 `_scheduled` 队列，将到时的 callback 放到 `_ready` 队列里
- 执行 `_ready` 里的每一个 callback

这一篇文章我们从用户的角度看了怎么创建事件循环，怎么让事件循环运行，怎么向事件循环注册 callback 以尽快或者将来调用。

然后我们从 Python 的源码看了事件循环的大概实现：当我们调用那些注册 callback 或者 `run_forever` 的时候，Python 内部具体做了什么。我们理解了事件循环做了什么以及怎么做的。

下一篇我们会介绍**协程**，