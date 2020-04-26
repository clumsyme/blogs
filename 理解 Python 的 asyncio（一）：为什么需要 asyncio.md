这不是我第一次尝试学习 Python 的 [`asyncio`](https://docs.python.org/3/library/asyncio.html) 模块了，之前每次尝试结果都是似懂非懂，再加上由于没有实践，过一段时间就又忘了那些基本概念。为什么这样？主要是 `asyncio` 模块涉及了太多了基本概念：

- 事件循环(event loop)
- 协程(coroutine)
- 协程函数
- 基于生成器的协程
- future
- 可等待对象(awaitable)
- 任务（task）
- handles

等等等等。就连 Armin Ronacher，Flask 框架作者，也写过一篇文章：[I don't understand Python's Asyncio](https://lucumr.pocoo.org/2016/10/30/i-dont-understand-asyncio/)，在文章最后总结道：

> Man that thing is complex and it keeps getting more complex. I do not have the mental capacity to casually work with asyncio. It requires constantly updating the knowledge with all language changes and it has tremendously complicated the language. It's impressive that an ecosystem is evolving around it but I can't help but get the impression that it will take quite a few more years for it to become a particularly enjoyable and stable development experience.

现在到了 2020 年，`asyncio` 模块随着 Python 迭代也在逐步改进。在 3.7+ 版本的 Python，使用 `asyncio` 看起来也简单了很多。

```py
import asyncio

async def main():
    print('Hello ...')
    await asyncio.sleep(1)
    print('... World')

asyncio.run(main())
```

所以，可能是时候来（再次尝试）理解 Python 的 `asyncio` 模块了。

## 为什么需要异步代码

### 同步 VS 异步

对于一个网页，或者任意的用户界面程序，影响用户体验的一个很重要因素就是延迟。比如我们在网页上运行了一个很耗时的 JavaScript 函数，这将会导致网页**卡住**，也就是这段时间内网页不会有任何反应。

尝试点击下边这个按钮来**卡住**页面。

<button id="laggy-button"> 卡住页面几秒钟</button>

<script>
    function fib(n) {
        if (n === 1 || n === 2) {
            return n
        }
        return fib(n-1) + fib(n-2)
    }
    let laggy_button = document.getElementById('laggy-button')
    laggy_button.addEventListener('click', function(e) {
        fib(44)
    })
</script>

可以发现，再点击之后，我们的鼠标好像失灵了一样，除了可以上下滚动，我们不能点击链接，不能复制文字。为了恢复鼠标的功能，我们**必须等待点击事件回调函数处理完成**，这需要花费好几秒钟。

再看下边的例子，点击按钮会发起一个网络请求，

<button id="fetch-button">点击发送网络请求</button>
<ul id="cpython-commits">

</ul>

<script>
let fetch_button = document.getElementById('fetch-button')
let commits_block = document.getElementById('cpython-commits')
fetch_button.addEventListener('click', async () => {
    commits_block.innerHTML = 'Fetching commits...'
    let response = await fetch('https://api.github.com/repos/python/cpython/commits')
    let result = await response.json()
    commits_block.innerHTML = ''
    for (let commit of result.slice(0, 3)) {
        let li = document.createElement('li')
        li.textContent = `${commit.commit.author.name}: ${commit.commit.message}`
        commits_block.appendChild(li)
    }
})
</script>

点击按钮之后，会发送一个请求，获取最新的 [CPython](https://github.com/python/cpython) commits 记录，并渲染到页面上。有时候网络请求返回很快，有时候比较慢，但是不管快慢，我们的页面从来不会卡住没响应。也就是说我们的页面不必去等待网络请求完成才恢复响应。

以上两个例子就是同步与异步的区别。

使用同步代码，意味着我们的程序必须按顺序一步一步执行。当前运行的代码会**阻塞**其他代码。如果某一块代码运行较慢，那其他部分代码要想得到运行只能等待这一块代码运行结束。

就像一个只有单车道的公路一样，即使你开着超级跑车，但是只要你前边有一个司机因为忙着干什么事在路上停了下来，你只能等待。

而使用异步代码，当程序运行一些需要大量时间的代码时，我们把它放到后台去运行。在前台，我们的代码将**不会被阻塞**。

那么什么叫把代码放到后台运行呢？以上述单车道公路为例，最简单的方法是我们再开辟一个车道。这个方案对比与程序就是使用**多线程**。

## 多线程

多线程可以让我们实现**并发**。程序开始运行的线程通常称为*主线程*，而其他线程可以称为 `Secondary Threads` 或者 `Worker Threads` 等等。以上边的同步按钮为例，我们可以把计算放到 [`Web Worker`](https://developer.mozilla.org/zh-CN/docs/Web/API/Web_Workers_API/Using_web_workers) 里，这样主线程就不会阻塞。当 Worker 计算完成后，它可以给主线程 [`postMessage`](https://developer.mozilla.org/zh-CN/docs/Web/API/Worker/postMessage) 来通知主线程处理计算结果。

但是线程不是免费的，它占用内存，需要占用操作系统调度。而且还面临着数据同步的问题。

### 多线程的问题

多个线程并不是多个程序，他们共享内存空间。如果多个线程同时去读取、修改一个数据，那就会造成数据不统一。

```py
def main():
    data = {"count": 1}

    def increase():
        count = data["count"]
        time.sleep(1)
        data["count"] = count + 1

    thread = Thread(target=increase, args=())
    thread.start()
    increase()
    print(data)
```

如上述例子，两个线程都对 `data['count']` 进行了 `+1` 操作，但是最终 `data['count']` 是 `2` ，只增加了 1。

为了解决数据不统一，我们需要多种数据同步手段，例如 [锁](https://docs.python.org/zh-cn/3/library/threading.html#lock-objects)，[信号量](https://docs.python.org/zh-cn/3/library/threading.html#semaphore-objects)，[栅栏](https://docs.python.org/zh-cn/3/library/threading.html#barrier-objects) 等。如果很多线程需要共享一个数据，我们可以使用一个锁，哪个线程获取了这个锁，它就可以操作该数据。操作完成之后，它会释放这个锁。这样就避免了数据错误的问题。

但是，如果一个线程获取了锁，其他线程如果也想操作该数据，它就必须等待持有锁的线程释放锁。

比如下图所示，两个线程 `o` 表示线程获取锁，`x` 表示线程释放锁，`.  `表示线程等待其他线程释放锁。

```
|o----------------x---------------|
|-----.            o--------x-----|
```

如果我们有 4 个线程，可能会出现下边的情形：

```
|o----------------x-------.  o-----x-----|
|-----.            o--------x------------|
|--------.                          o----|
|----------.                             |
```

很多线程需要访问共享数据，结果时间都浪费到了等待锁被释放了。比如上图，第四个线程从等待到最后一直都没有获取到锁。

如果有了更多的线程，更多的共享数据（意味着更多的锁），每个线程需要内存等资源，结果就是高的内存占用和长时间的等待锁释放，程序也会变慢。更多的锁可能带来新的问题，例如[死锁](https://zh.wikipedia.org/zh-cn/%E6%AD%BB%E9%94%81)。

### 全局解释器锁

Python 的多线程还需要面临另一个问题，[全局解释器锁](https://zh.wikipedia.org/wiki/%E5%85%A8%E5%B1%80%E8%A7%A3%E9%87%8A%E5%99%A8%E9%94%81)，它使得即使在多 CPU 机器上，一个 Python 进程在同一时刻只有一个线程在运行，使得多个 CPU 没有被充分利用。

## I/O

所以，编写多线程 Python 代码有很多注意事项；而且多线程还不能充分利用 CPU。还有一个问题也会导致这个问题：Input/Output，也就是 I/O，如文件读取、网络请求。通常一个 I/O 事件会花费较长时间来完成，如果线程发起一个 I/O 操作，然后一直等待操作完成，这段时间 CPU 就完全闲置了。如果我们可以让 Python 在这段时间内去处理点别的任务，让 I/O 事件在后台进行不是更好吗？

这也就是 `asyncio` 的基本思想：通过异步 I/O、使用协程实现并发来最大化单线程的利用率。

异步 I/O 接口不会阻塞线程，如果我们发起一个网络请求，这个调用会立刻返回，我们就可以接着处理其他任务浏览。那么我们怎么知道什么时候请求完成了呢？

## 基于 I/O 多路复用的并发

基于 I/O 多路复用的并发的基本思想就是，提供一个 `select` 函数，当调用时，系统会挂起进程，只有当一个或者多个 I/O 事件发生之后，才将控制返回进程。就像下边一样：

- 如果集合 {0, 4} 中任何描述符准备好了读时返回
- 如果集合 {1,2,7} 中任意描述符准备好了写时返回
- 如果等待了 123 秒，就超时返回

有了 `select` 函数，我们就可以用一个简单的循环：

```py
while True:
    event_list = select()
    for event in event_list:
        process(event)
```

### `select`

现代的操作系统都提供了各种系统调用，可以让我们监测一些文件的状态，除了 [`select`](http://man7.org/linux/man-pages/man2/select.2.html)，现在有性能更好的 [kqueue](https://www.freebsd.org/cgi/man.cgi?kqueue)、[epoll](https://zh.wikipedia.org/wiki/Epoll)、[IOCP(Windows)](https://docs.microsoft.com/zh-cn/windows/win32/fileio/i-o-completion-ports?redirectedfrom=MSDN)等。

Python 在这些基础上，抽象了一个高层的 [`DefaultSelector`](https://docs.python.org/zh-cn/3/library/selectors.html#selectors.DefaultSelector) 类，会根据平台的不同选择不同的实现。该类提供一个统一的 `select` 方法：

```py
abstractmethod select(timeout=None)
```

- 如果 timeout 参数大于 0，那么 `select` 函数会阻塞 `timeout` 时间，到时之后会将控制交还给程序。
- 如果 timeout 小于等于 0，那么 `select` 函数将直接返回当前准备好的 fileobj，并且不阻塞。
- 如果 timeout 是 `None`，那么 `select` 函数会一直阻塞，直到一个被监测的文件状态变为 ready。

有了异步 I/O 接口、`select` 函数，我们就可以创建 `asyncio` 应用的核心：**事件循环**。

下一章我们来探索 `asyncio` 中的事件循环。

## 参考：

---

- [I don't understand Python's Asyncio - Armin Ronacher's Thoughts and Writings](https://lucumr.pocoo.org/2016/10/30/i-dont-understand-asyncio/)
- [asyncio — Asynchronous I/O](https://docs.python.org/3/library/asyncio.html)
- [threading — Thread-based parallelism](https://docs.python.org/3/library/threading.html)
- [Global Interpreter Lock - Python Wiki](https://wiki.python.org/moin/GlobalInterpreterLock)
- [select(2) - Linux manual page](http://man7.org/linux/man-pages/man2/select.2.html)
- [kqueue - FreeBSD Manual Pages](https://www.freebsd.org/cgi/man.cgi?kqueue)
- [I/O Completion Ports](https://docs.microsoft.com/zh-cn/windows/win32/fileio/i-o-completion-ports?redirectedfrom=MSDN)
- [selectors — High-level I/O multiplexing](https://docs.python.org/3/library/selectors.html)
- [深入理解计算机系统](https://book.douban.com/subject/26912767/)