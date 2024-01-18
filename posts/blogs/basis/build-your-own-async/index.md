# 深入理解 Asyncio，从零开始构建你自己的并发调度器


Python 并发实现的两种方式：

1. 基于回调的实现
2. 基于生成器(协程)的实现

下面将基于两个简单的例子分别介绍如何从 0 开始以两种方式实现自己的异步框架：

1. 相互独立的函数的并发
2. 生产者消费者模式的并发

## 1. 同步方式

有两个相互独立的函数：

1. `def countdown(n)`：从 n 开始往下打印到 0
2. `def countup(n)`：从 0 开始向上打印到 n - 1

```python
import time


def countdown(n):
    while n > 0:
        print('Down', n)
        time.sleep(1)
        n -= 1


def countup(stop):
    x = 0
    while x < stop:
        print('Up', x)
        time.sleep(1)
        x += 1
```

以同步方式运行：

```python
countdown(5)
countup(5)
```

输出：

```bash
Down 5
Down 4
Down 3
Down 2
Down 1
Up 0
Up 1
Up 2
Up 3
Up 4
```

我们可以以多线程的方式进行并发调度：

```python
import threading

threading.Thread(target=countdown, args=(5,)).start()
threading.Thread(target=countup, args=(5,)).start()
```


## 2. 基于回调的并发调度

如果不考虑多线程的方式，我们来实现自己的并发调度器。
首先，我们来实现一个调度器：

```python
class Scheduler:
    def __init__(self):
        self.ready = deque()  # 准备好执行的 function 队列
        self.sleeping = []  # 等待执行的 function 优先级队列

    def call_soon(self, func):
        """将 func 加入 ready 队列"""
        self.ready.append(func)

    def call_later(self, delay, func):
        """将 func 加入 sleeping 队列"""
        deadline = time.time() + delay  # 计算执行时间，将其作为优先级
        heapq.heappush(self.sleeping, (deadline, func))

    def run(self):
        while self.ready or self.sleeping:
            if not self.ready:
                # 取出等待队列中优先级最高的(执行时间最近的)，放入 ready 队列
                deadline, func = heapq.heappop(self.sleeping)
                delta = deadline - time.time()
                if delta > 0:
                    time.sleep(delta)
                self.ready.append(func)

            while self.ready:  # 从 ready 队列中取出一个函数执行
                func = self.ready.popleft()
                func()
```

通过上面的代码，我们实现了一个基本的调度器，可以实现异步调度的功能：

- 要立即运行一个函数，我们并不直接调用，而是调用 `call_soon` 方法，将其加入 `ready` 队列，等待 `run` 方法调度
- 要延迟运行一个函数，调用 `call_later` 方法

其中 `run` 方法是调度的核心，它的执行逻辑如下：

- 当 `ready` 队列或 `sleeping` 队列有待执行函数时，循环取出并执行
- 首先从 `ready` 队列取出并执行
- `ready` 队列为空后，从 `sleeping` 队列取出最先执行的函数，放入 `ready` 队列

但我们的调度器还有点问题，如果两个函数的 `deadline` 一样，则会报错:

```bash
TypeError: '<' not supported between instances of 'function' and 'function'
```

这是由于 `heappush` 方法将首先比较 `(deadline, func)` 这个 tuple 中的第一个元素 `deadline`，当 `dealine` 相同的时候，会比较第二个元素，但第二个元素是一个函数，不能做比较。

要解决这个问题，我们可以在 `(deadline, func)` 这个 tuple 中加一个自增的量 `sequence`，变成 `(deadline, sequence, func)`，这样如果两个函数的 `deadline` 相同，则会依入队顺序排序

完整代码如下：

```python
class Scheduler:
    def __init__(self):
        self.ready = deque()  # 准备好执行的 function 队列
        self.sleeping = []  # 等待执行的 function 优先级队列
        self.sequence = 0  # 自增量，避免 heappush 抛异常

    def call_soon(self, func):
        """将 func 加入 ready 队列"""
        self.ready.append(func)

    def call_later(self, delay, func):
        """将 func 加入 sleeping 队列"""
        self.sequence += 1
        deadline = time.time() + delay  # 计算执行时间，将其作为优先级
        heapq.heappush(self.sleeping, (deadline, self.sequence, func))

    def run(self):
        while self.ready or self.sleeping:
            if not self.ready:
                # 取出等待队列中优先级最高的(执行时间最近的)，放入 ready 队列
                deadline, _, func = heapq.heappop(self.sleeping)
                delta = deadline - time.time()
                if delta > 0:
                    time.sleep(delta)
                self.ready.append(func)

            while self.ready:  # 从 ready 队列中取出一个函数执行
                func = self.ready.popleft()
                func()
```

这时，我们必须修改我们的 `countup` 和 `countdown` 方法：

```python
def countdown(n):
    if n > 0:
        print('Down', n)
        # time.sleep(4)  # 不要再使用阻塞的代码
        sched.call_later(4, lambda: countdown(n - 1))  # 要传递函数，而不能传递调用


def countup(stop):
    def _run(x):
        if x < stop:
            print('Up', x)
            # time.sleep(1) # 不要再使用阻塞的代码
            sched.call_later(1, lambda: _run(x + 1))

    _run(0)
```

- 去掉循环，改成回调，每次打印完成后，把下一个函数送入队列
- 阻塞的代码`time.sleep()` 去掉，改为 `call_later()` 控制调度时机


调度函数，启动循环：

```python
sched.call_soon(lambda: countdown(5))
sched.call_soon(lambda: countup(20))
sched.run()
```

输出如下：

```bash
Down 5
Up 0
Up 1
Up 2
Up 3
Down 4
...
```

可以看出，的确并发的运行了我们的两个函数。

## 3. 生产者-消费者模型

### 3.1 多线程调度

下面我们来看看多线程的生产者-消费者调度，代码如下：

```python
import queue
import threading
import time


def producer(q, count):
    for n in range(count):
        print('Producing', n)
        q.put(n)
        time.sleep(1)

    print('Producer done')
    q.put(None)  # 关闭信号


def consumer(q):
    while True:
        item = q.get()
        if item is None:
            break
        print('Consuming', item)
    print('Consumer done')


q = queue.Queue()  # Thread-safe queue
threading.Thread(target=producer, args=(q, 10)).start()
threading.Thread(target=consumer, args=(q,)).start()
```

这个实现很简单，生产者每隔一秒往队列 put 一个消息，消费者从队列取出并打印，不再赘述

### 3.2 回调调度

下面我们来看看，如何使用我们的 `Scheduler` 来调度生产者消费者，从而实现并发。

生产者-消费者模型相比独立的两个函数的调度多了一个队列来传递消息，因此我们必须要自己实现一个满足我们的回调的设计的队列，代码如下：

```python
sched = Scheduler()


class QueueClosed(Exception):
    pass


class Result:
    """用于消息和异常的封装"""
    def __init__(self, value=None, exc=None):
        self.value = value
        self.exc = exc

    def result(self):
        if self.exc:
            raise self.exc
        else:
            return self.value


class AsyncQueue:
    """实现回调调度的队列"""
    def __init__(self):
        self.items = deque()
        self.waiting = deque()  # 等待消息的消费者
        self._closed = False  # 队列是否已关闭

    def close(self):
        """关闭队列，让所有等待的消费者立即执行"""
        self._closed = True
        if self.waiting and not self.items:
            for func in self.waiting:
                sched.call_soon(func)

    def put(self, item):
        """传入消息，如果有等待的消费者，立即取出一个并执行"""
        if self._closed:
            raise QueueClosed()
        self.items.append(item)
        if self.waiting:
            func = self.waiting.popleft()
            sched.call_soon(func)

    def get(self, callback):
        """取出消息，并调用 callback，如果没有消息，则将消费者 (callback) 放入等待队列"""
        if self.items:  # 有消息，立即调度
            sched.call_soon(lambda: callback(Result(self.items.popleft())))
        else:
            if self._closed:  # 队列已关闭，Result 抛出一个异常
                sched.call_soon(lambda: callback(Result(exc=QueueClosed())))

            else:  # 没有 item 时，不要阻塞，将自己放入等待队列，直到下一个消息到达，
                self.waiting.append(lambda: self.get(callback))
```

我们实现了一个 `Result` 类，以封装队列关闭的异常和取出的消息，使其有统一的接口，具体用法将在下面看到

我们实现了一个 `AsnyncQueue` 类，以配合我们的 `Scheduler` 完成异步调度：

- `close` 方法用于关闭队列，并把所有等待的消费者立即调度
- `put` 方法将消息传入队列，传入的时候，如果有在队列中等待的消费者，因把它立即调度
- `get` 方法将消息取出，并传入 `callback` 进行调用，这里是我们回调的重点。回调的基本思想是：传入回调函数，等待资源就绪后，立即调用回调函数，因此我们要把消费者传入 `get` 方法，而不是直接返回一条消息，供调用者使用

同样，我们也要修改上一节的生产者和消费者函数，由于我们要采用回调的方式，生产者和消费者中的循环都要改为回调，在消费者中，还要把自己传入队列，等待资源就绪后调用。

修改后的生产者和消费者函数如下：

```python
def producer(q, count):
    def _run(n):
        if n < count:
            print('Producing', n)
            q.put(n)
            sched.call_later(1, lambda: _run(n + 1))  # 取消循环，改为回调
        else:
            print('Producer done')
            q.close()

    _run(0)


def consumer(q):
    def _consume(result):
        try:
            item = result.result()  # 由于对消息进行了封装，要通过该方法取出消息或抛出异常
            print('Consuming', item)
            sched.call_soon(lambda: consumer(q))
        except QueueClosed:
            print('Consumer done')

    q.get(callback=_consume)


q = AsyncQueue()
sched.call_soon(lambda: producer(q, 10))
sched.call_soon(lambda: consumer(q))
sched.run()
```

由于采用了回调，很难理解这段代码，我们不妨举例说明：

- 当 `producer` 首次被调用时，将运行 `_run` 函数，参数 `n = 0`，接着我们把 `0` 放入 `AsyncQueue` 的队列中，再接着往 `Scheduler` 传入下一个等待调度的函数 `_run`，参数 `n + 1`，这里比较容易理解
- 当 `consumer` 首次被调用时，我们调用 `AsyncQueue` 的 `get` 方法，传入 `callback=_consume`，由于已经有一条消息 `0`，`_consume` 立即运行(并不一定是立即，取决于 `Scheduler` 中 `ready` 队列的状态，当然此时是立即执行的，因为在此之前 `ready` 队列是空的)，`result.result()` 返回 `0`，因此立即打印 `Consuming 0`，并调用 `sched.call_soon(lambda: consumer(q))` 再次将自己加入 `ready` 队列，将立即被调度
- 这时由于 `producer` 会等待 1s 后调用，此时还没有消息，因此在 `AsyncQueue` 的 `get` 方法中，会把 `_consume` 加入 `waiting` 队列
- 直到下一个 `_run` 运行，将 `1` 加入 `items` 中，并在 `AsyncQueue` 的 `put` 方法中将上一把加入 `waiting` 队列的消费唤醒并调度
- 重复以上过程，即实现了生产者-消费者的异步调度

注意：当我们使用基于回调的异步调度时，不能在我们的生产者或消费者中引入**阻塞**的代码，否则会在调度器的 `run()` 方法中阻塞住整个循环，并且要把所有的**循环**，改为回调形式

我们可以看出，基于回调的并发调度有诸多缺点：

- 代码相比同步形式，结构上有了很大改变
- 正因此改变使其变得难以理解
- 如果回调层数增加，将更一步的难以阅读和理解

## 4. 基于生成器的并发调度

正是由于回调难以理解，代码相比同步方式改动太大，我们将实现基于生成器(协程)的并发调度。

对于生成器不了解的童鞋建议 Google 一下再来接着看

### 4.1 相互独立的函数调度

#### 4.1.1 基本实现

和之前一样，我们也要实现一个调度器，先看一下最基本的实现:

```python
import time
from collections import deque


class Scheduler:
    def __init__(self):
        self.ready = deque()
        self.current = None  # Current executing generator

    def new_task(self, func):
        self.ready.append(func)

    def run(self):
        while self.ready:
            self.current = self.ready.popleft()
            try:
                next(self.current)
                if self.current:
                    self.ready.append(self.current)
            except StopIteration:
                pass


sched = Scheduler()


def countdown(n):
    while n > 0:
        print('Down', n)
        time.sleep(1)
        yield
        n -= 1


def countup(stop):
    x = 0
    while x < stop:
        print('Up', x)
        time.sleep(1)
        yield
        x += 1


sched.new_task(countdown(5))
sched.new_task(countup(5))
sched.run()
```

这是一个基本结构，我们通过 `new_task` 方法将生成器放入队列，调用 `run()` 方法运行调度器后，循环的从 `ready` 队列取出生成器，调用 `next` 方法使其运行到下一个 `yield` 处，然后我们将生成器放回 `ready` 队列等待下一次调用，直到生成器抛出 `StopIteration` 异常。

由于我们的生成器中有阻塞代码 `time.sleep`，他并没有真正并发起来，但我们的调度器的基本结构就是这样的：循环调度生成器。稍后我们实现一个不阻塞的 `sleep` 即可实现并发(类似的在基于回调的调度中是由 `call_later` 实现的)。

此处的关键点在于：我们的 `countdown` 和 `countup` 函数和原来同步的函数除了多了一个 `yield` 以外没什么区别，相当容易阅读和理解。

#### 4.1.2 更优雅的写法

毕竟多了个 `yield` 还是不太好看，他唯一的功能就是暂停函数的执行，交出控制权。

下面我们将使用协程来代替生成器，简单来讲使用 `async def` 定义的函数就是一个协程，其中可以使用 `await` 调用其他协程或 `Awaitable` 对象，但归根结底，我们还是需要一个 `yield` 来暂停函数执行，交出控制权。

实现方式如下：

```python
import time
from collections import deque


class Awaitable:
    """Awaitable 对象"""
    def __await__(self):
        """Awaitable 对象中，__await__ 方法要返回一个生成器"""
        yield


def switch():
    return Awaitable()


class Scheduler:
    def __init__(self):
        self.ready = deque()
        self.current = None  # Current executing generator

    def new_task(self, func):
        self.ready.append(func)

    def run(self):
        while self.ready:
            self.current = self.ready.popleft()
            try:
                # next(self.current)
                self.current.send(None)
                if self.current:
                    self.ready.append(self.current)
            except StopIteration:
                pass


sched = Scheduler()


async def countdown(n):
    while n > 0:
        print('Down', n)
        time.sleep(1)
        await switch()
        n -= 1


async def countup(stop):
    x = 0
    while x < stop:
        print('Up', x)
        time.sleep(1)
        await switch()
        x += 1


sched.new_task(countdown(5))
sched.new_task(countup(5))
sched.run()
```

我们把生成器改为了协程，并在其中 `await` 一个 `Awaitable` 对象，来暂停协程运行。当运行到 `await switch()` 时，会进入 `Awaitable` 的 `__await__` 方法，执行到 `yield` 时，暂停函数执行

由于改用了协程，因此原本的唤醒方式 `next(generator)` 要改为 `corotine.send(None)`，即 `self.current.send(None)` 

#### 4.1.3 真正并发起来

我们将实现非阻塞的 `sleep` 方法来实现真正的并发，代码如下：

```python
import heapq
import time
from collections import deque


class Awaitable:

    def __await__(self):
        yield


def switch():
    return Awaitable()


class Scheduler:
    def __init__(self):
        self.ready = deque()
        self.sleeping = []
        self.current = None  # 正在运行的 coroutine
        self.sequence = 0

    def new_task(self, coro):
        self.ready.append(coro)

    async def sleep(self, delay):
        """将当前协程加入 sleeping 队列，并从 ready 队列删除"""
        deadline = time.time() + delay
        self.sequence += 1
        heapq.heappush(self.sleeping, (deadline, self.sequence, self.current))
        self.current = None  # 让当前 coroutine 不要再被加入 ready 队列
        await switch()  # Switch tasks

    def run(self):
        while self.ready or self.sleeping:
            if not self.ready:  # 尝试从 sleeping 队列唤醒一个 coroutine
                deadline, _, coro = heapq.heappop(self.sleeping)  # 取出最近的一个协程
                delta = deadline - time.time()
                # 如果还不到时间，就阻塞等待，因为这个是最早要执行的协程了，可以阻塞等待
                if delta > 0:
                    time.sleep(delta)
                self.ready.append(coro)

            self.current = self.ready.popleft()
            # Drive as a generator
            try:
                self.current.send(None)
                if self.current:
                    self.ready.append(self.current)
            except StopIteration:
                pass


sched = Scheduler()


async def countdown(n):
    while n > 0:
        print('Down', n)
        await sched.sleep(4)
        n -= 1


async def countup(stop):
    x = 0
    while x < stop:
        print('Up', x)
        await sched.sleep(1)
        x += 1


sched.new_task(countdown(5))
sched.new_task(countup(20))
sched.run()
```

输出：

```bash
Down 5
Up 0
Up 1
Up 2
Up 3
Down 4
Up 4
...
```

在 `countdown` 和 `countup` 中调用 `shced.sleep(1)` 类似我们调用 `asyncio.sleep(1)` 这里已经有一些 `asyncio` 的影子了。

当我们的调度器运行到 `sleep` 方法时，`self.corrent` 就是当前运行的协程 `countdown` 或 `countup`，进入 `sleep` 后，将其加入 `sleeping` 队列，并将 `self.current` 置为 `None` 以使其不再被加入 `ready` 队列

至此，我们的基于协程的并发调度已经完成。可以看出，我们原本的代码逻辑完全没有变化，只是将普通函数变成了协程(`async` 和 `await` 语法)，并把 `time.sleep()` 换成了我们的非阻塞的 `sched.sleep()`

### 4.2 生产者-消费者模型调度

从[#第 3 节](#3-生产者-消费者模型) 我们可以看出，要实现生产者-消费者模型，仅仅需要实现一个队列即可。我们将以类似的方式实现一个队列：

```python
class QueueClosed(Exception):
    pass


class AsyncQueue:
    def __init__(self):
        self.items = deque()
        self.waiting = deque()
        self._closed = False

    def close(self):
        self._closed = True
        if self.waiting:
            sched.ready.append(self.waiting.popleft())

    async def put(self, item):
        if self._closed:
            raise QueueClosed()

        self.items.append(item)
        if self.waiting:
            sched.ready.append(self.waiting.popleft())

    async def get(self):
        while not self.items:
            if self._closed:
                raise QueueClosed()

            self.waiting.append(sched.current)  # Put myself to sleep
            sched.current = None  # "Disappear"
            await switch()  # Switch to another task

        return self.items.popleft()
```

同[#第 3 节](#3-生产者-消费者模型)类似，不再赘述，要注意在 `get` 方法中，与上一节的 `sleep` 类似，将协程放入等待队列后，要把调度器的 `current` 置为 `None` 避免其再次被加入 `ready` 队列

> 注意，此处 `put` 方法完全可以不是协程，此处这样写只是为了和 `get` 方法对应。

同样，我们的生产者-消费者代码也不需要太大改动，如下：

```python
async def producer(q, count):
    for n in range(count):
        print('Producing', n)
        await q.put(n)
        await sched.sleep(1)

    print('Producer done')
    q.close()  # "Sentinel" to shut down


async def consumer(q):
    try:
        while True:
            item = await q.get()
            if item is None:
                break
            print('Consuming', item)
    except QueueClosed:
        print('Consumer done')
```

由于我们没有回调了，我们不再需要把 `consumer` 传入 `get` 方法，在其中以回调的方式调用，而是直接把消息返回出来。我们的代码相比同步的代码除了 `async` 和 `await` 没有太大变化，易于阅读和理解

完整代码如下：

```python
import heapq
import time
from collections import deque


class Scheduler:
    def __init__(self):
        self.ready = deque()
        self.sleeping = []
        self.current = None  # Current executing generator
        self.sequence = 0

    def new_task(self, coro):
        self.ready.append(coro)

    async def sleep(self, delay):
        # The current "coroutine" wants to sleep. How?
        deadline = time.time() + delay
        self.sequence += 1
        heapq.heappush(self.sleeping, (deadline, self.sequence, self.current))
        self.current = None  # 让当前 coroutine 不要再被加入 ready 队列
        await switch()  # Switch tasks

    def run(self):
        while self.ready or self.sleeping:
            if not self.ready:
                deadline, _, coro = heapq.heappop(self.sleeping)
                delta = deadline - time.time()
                if delta > 0:
                    time.sleep(delta)
                self.ready.append(coro)

            self.current = self.ready.popleft()
            # Drive as a generator
            try:
                # next(self.current)
                self.current.send(None)  # 实际执行 coroutine 的位置
                if self.current:
                    self.ready.append(self.current)
            except StopIteration:
                pass


sched = Scheduler()  # Background scheduler object


class Awaitable:

    def __await__(self):
        yield


def switch():
    return Awaitable()


class QueueClosed(Exception):
    pass


class AsyncQueue:
    def __init__(self):
        self.items = deque()
        self.waiting = deque()
        self._closed = False

    def close(self):
        self._closed = True
        if self.waiting:
            sched.ready.append(self.waiting.popleft())

    async def put(self, item):
        if self._closed:
            raise QueueClosed()

        self.items.append(item)
        if self.waiting:
            sched.ready.append(self.waiting.popleft())

    async def get(self):
        while not self.items:
            if self._closed:
                raise QueueClosed()

            self.waiting.append(sched.current)  # Put myself to sleep
            sched.current = None  # "Disappear"
            await switch()  # Switch to another task

        return self.items.popleft()


async def producer(q, count):
    for n in range(count):
        print('Producing', n)
        await q.put(n)
        await sched.sleep(1)

    print('Producer done')
    q.close()  # "Sentinel" to shut down


async def consumer(q):
    try:
        while True:
            item = await q.get()
            if item is None:
                break
            print('Consuming', item)
    except QueueClosed:
        print('Consumer done')


q = AsyncQueue()
sched.new_task(producer(q, 10))
sched.new_task(consumer(q))
sched.run()
```

我们再来讲一讲调度器启动后的执行过程：

- `producer` 通过 `await q.put(n)` 传入一条消息 `0`，然后运行 `await sched.sleep(1)` 进入 `sleep` 方法，将 `producer` 协程加入 `sleeping` 队列
- `consumer` 取出 `0` 并打印，在 `run` 方法中再次将自己加入 `ready` 队列，此时 `producer` 还未唤醒，`consumer` 将立即执行
- `consumer` 执行到 `await q.get()`，但 `q` 中还没有消息，因此将 `consumer` 加入 `waiting` 队列
- `producer` 唤醒后，产生一条消息 `1`，并唤醒 `consumer`
- 以此往复，直到 `producer` 关闭队列，`consumer` 打印最后一条消息，再次调度 `consumer` 时，`get` 中抛出异常，结束

## 5. 更类似 Asyncio 的方式

在 `asyncio` 中并不简单的是我们上述的完全基于协程的调度方式，也不是基于回调的。而是结合两者，通过对协程进行封装，最后交由回调进行调度。这也正是其难以理解之处。

下面我们将实现结合回调和协程的调度，`Scheduler` 如下：

```python
class Task:
    """实现协程的封装，使其可以像回调一样被调度"""
    def __init__(self, coro):
        self.coro = coro  # "Wrapped coroutine"

    # Make it look like a callback
    def __call__(self, *args, **kwargs):
        try:
            # Driving the coroutine as before
            sched.current = self
            self.coro.send(None)
            if sched.current:
                sched.ready.append(self)
        except StopIteration:
            pass
            
class Scheduler:
    def __init__(self):
        self.ready = deque()  # Functions ready to execute
        self.sleeping = []  # Sleeping functions
        self.sequence = 0
        self.current = None

    def call_soon(self, func):
        self.ready.append(func)

    def call_later(self, delay, func):
        self.sequence += 1
        deadline = time.time() + delay  # Expiration time
        # Priority queue
        heapq.heappush(self.sleeping, (deadline, self.sequence, func))

    def run(self):
        while self.ready or self.sleeping:
            if not self.ready:
                # Find the nearest deadline
                deadline, _, func = heapq.heappop(self.sleeping)
                delta = deadline - time.time()
                if delta > 0:
                    time.sleep(delta)
                self.ready.append(func)

            while self.ready:
                func = self.ready.popleft()
                func()

    def new_task(self, coro):
        self.ready.append(Task(coro))  # Wrapped coroutine

    async def sleep(self, delay):
        self.call_later(delay, self.current)
        self.current = None
        await switch()
```

实现方式如下：

1. 在基于回调的调度器基础上添加 `new_task` 方法用于将协程加入调度器，并实现 `sleep` 方法，还多了一个 `self.current` 变量，用于记录当前协程
2. 增加 `Task` 类，使协程像普通函数一样被调度：通过 `new_task` 传入的协程，都被封装在 `Task` 中，但 `Task` 被调用时，实际上使用 `self.coro.send(None)` 来启动协程

其他地方，我们无需做更多更改，我们既可以写一个协程来让其调度，也可以把我们的函数协程回调形式，让其调度，完整代码如下：

```python
import heapq
import time
from collections import deque


class Scheduler:
    def __init__(self):
        self.ready = deque()  # Functions ready to execute
        self.sleeping = []  # Sleeping functions
        self.sequence = 0
        self.current = None

    def call_soon(self, func):
        self.ready.append(func)

    def call_later(self, delay, func):
        self.sequence += 1
        deadline = time.time() + delay  # Expiration time
        # Priority queue
        heapq.heappush(self.sleeping, (deadline, self.sequence, func))

    def run(self):
        while self.ready or self.sleeping:
            if not self.ready:
                # Find the nearest deadline
                deadline, _, func = heapq.heappop(self.sleeping)
                delta = deadline - time.time()
                if delta > 0:
                    time.sleep(delta)
                self.ready.append(func)

            while self.ready:
                func = self.ready.popleft()
                func()

    def new_task(self, coro):
        self.ready.append(Task(coro))  # Wrapped coroutine

    async def sleep(self, delay):
        self.call_later(delay, self.current)
        self.current = None
        await switch()


class Task:
    """实现协程的封装，使其可以像回调一样被调度"""

    def __init__(self, coro):
        self.coro = coro  # "Wrapped coroutine"

    # Make it look like a callback
    def __call__(self, *args, **kwargs):
        try:
            # Driving the coroutine as before
            sched.current = self
            self.coro.send(None)
            if sched.current:
                sched.ready.append(self)
        except StopIteration:
            pass


sched = Scheduler()


class Awaitable:

    def __await__(self):
        yield


def switch():
    return Awaitable()


class QueueClosed(Exception):
    pass


class AsyncQueue:
    def __init__(self):
        self.items = deque()
        self.waiting = deque()
        self._closed = False

    def close(self):
        self._closed = True
        if self.waiting:
            sched.ready.append(self.waiting.popleft())

    async def put(self, item):
        if self._closed:
            raise QueueClosed()

        self.items.append(item)
        if self.waiting:
            sched.ready.append(self.waiting.popleft())

    async def get(self):
        while not self.items:
            if self._closed:
                raise QueueClosed()

            # Wait...
            self.waiting.append(sched.current)  # Put myself to sleep
            sched.current = None  # "Disappear"
            await switch()  # Switch to another task

        return self.items.popleft()


async def producer(q, count):
    for n in range(count):
        print('Producing', n)
        await q.put(n)
        await sched.sleep(1)

    print('Producer done')
    q.close()  # "Sentinel" to shut down


async def consumer(q):
    try:
        while True:
            item = await q.get()
            if item is None:
                break
            print('Consuming', item)
    except QueueClosed:
        print('Consumer done')


q = AsyncQueue()
sched.new_task(producer(q, 10))
sched.new_task(consumer(q))
sched.run()


def countdown(n):
    if n > 0:
        print('Down', n)
        sched.call_later(4, lambda: countdown(n - 1))


def countup(stop):
    def _run(x):
        if x < stop:
            print('Up', x)
            sched.call_later(1, lambda: _run(x + 1))

    _run(0)


sched.call_soon(lambda: countdown(5))
sched.call_soon(lambda: countup(20))
sched.run()
```

## 6. 并发 I/O

基于我们上述的基于回调组合基于协程的调度方式，我们可以为其增加并发 I/O 的实现，此处实现一个并发 socket 为例。

代码如下：

```python
import heapq
import socket
import time
from collections import deque

from select import select


class Scheduler:
    def __init__(self):
        self.ready = deque()  # Functions ready to execute
        self.sleeping = []  # Sleeping functions
        self.sequence = 0
        self.current = None
        self._read_waiting = {}
        self._write_waiting = {}

    def call_soon(self, func):
        self.ready.append(func)

    def call_later(self, delay, func):
        self.sequence += 1
        deadline = time.time() + delay  # Expiration time
        # Priority queue
        heapq.heappush(self.sleeping, (deadline, self.sequence, func))

    def read_wait(self, fileno, func):
        # Trigger func() when fileno is readable
        self._read_waiting[fileno] = func

    def write_wait(self, fileno, func):
        # Trigger func() when fileno is writeable
        self._write_waiting[fileno] = func

    def run(self):
        while self.ready or self.sleeping or self._read_waiting or self._write_waiting:
            if not self.ready:
                # Find the nearest deadline
                if self.sleeping:
                    deadline, _, func = self.sleeping[0]
                    timeout = deadline - time.time()
                    if timeout < 0:
                        timeout = 0
                else:
                    timeout = None  # Wait forever

                # Wait for I/O and sleep
                can_read, can_write, _ = select(self._read_waiting, self._write_waiting,
                                                [], timeout)
                for fd in can_read:
                    self.ready.append(self._read_waiting.pop(fd))
                for fd in can_write:
                    self.ready.append(self._write_waiting.pop(fd))

                # Check for sleeping tasks
                now = time.time()
                while self.sleeping:
                    if now > self.sleeping[0][0]:
                        self.ready.append(heapq.heappop(self.sleeping)[2])
                    else:
                        break

            while self.ready:
                func = self.ready.popleft()
                func()

    def new_task(self, coro):
        self.ready.append(Task(coro))  # Wrapped coroutine

    async def sleep(self, delay):
        self.call_later(delay, self.current)
        self.current = None
        await switch()

    async def recv(self, sock, maxbytes):
        self.read_wait(sock, self.current)
        self.current = None
        await switch()
        return sock.recv(maxbytes)

    async def send(self, sock, data):
        self.write_wait(sock, self.current)
        self.current = None
        await switch()
        return sock.send(data)

    async def accept(self, sock):
        self.read_wait(sock, self.current)
        self.current = None
        await switch()
        return sock.accept()


class Task:
    def __init__(self, coro):
        self.coro = coro  # "Wrapped coroutine"

    # Make it look like a callback
    def __call__(self, *args, **kwargs):
        try:
            # Driving the coroutine as before
            sched.current = self
            self.coro.send(None)
            if sched.current:
                sched.ready.append(self)
        except StopIteration:
            pass


sched = Scheduler()


class Awaitable:

    def __await__(self):
        yield


def switch():
    return Awaitable()


class QueueClosed(Exception):
    pass


class AsyncQueue:
    def __init__(self):
        self.items = deque()
        self.waiting = deque()
        self._closed = False

    def close(self):
        self._closed = True
        if self.waiting:
            sched.ready.append(self.waiting.popleft())

    async def put(self, item):
        if self._closed:
            raise QueueClosed()

        self.items.append(item)
        if self.waiting:
            sched.ready.append(self.waiting.popleft())

    async def get(self):
        while not self.items:
            if self._closed:
                raise QueueClosed()

            # Wait...
            self.waiting.append(sched.current)  # Put myself to sleep
            sched.current = None  # "Disappear"
            await switch()  # Switch to another task

        return self.items.popleft()


async def producer(q, count):
    for n in range(count):
        print('Producing', n)
        await q.put(n)
        await sched.sleep(1)

    print('Producer done')
    q.close()  # "Sentinel" to shut down


async def consumer(q):
    try:
        while True:
            item = await q.get()
            if item is None:
                break
            print('Consuming', item)
    except QueueClosed:
        print('Consumer done')


async def tcp_server(addr):
    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.bind(addr)
    sock.listen(1)
    while True:
        client, addr = await sched.accept(sock)
        sched.new_task(echo_handler(client))


async def echo_handler(sock):
    while True:
        data = await sched.recv(sock, 10000)
        if not data:
            break

        await sched.send(sock, b'Got:' + data)
    print('Connection closed')
    sock.close()


q = AsyncQueue()
sched.new_task(producer(q, 100))
sched.new_task(consumer(q))

sched.new_task(tcp_server(('', 30000)))

sched.run()
```

下面我们讨论下重点部分：

- `read_wait` 和 `write_wait` 方法将准备读写的文件句柄和准备调度的方法写入等待字典 `_read_waiting` 和 `_write_waiting`
- `recv`、`send` 和 `accept` 方法将对应的 socket fd 加入等待字典中
- `run` 方法除了遍历 `ready` 和 `sleeping` 队列，还要遍历 `_read_waiting` 和 `_write_waiting` 字典
- `run` 方法中通过 `can_read, can_write, _ = select(self._read_waiting, self._write_waiting, [], timeout)` 找出准备好的 `fd`，并唤醒 `fd` 对应的方法，即 `recv`、`send` 和 `accept`
- 在 `tcp_server` 中调用 `accept` 接受 socket 连接(异步的、并发的)，当由连接进入时，将 `echo_handler` 加入调度器，对当前 `client` 服务，这里没有阻塞，因此会继续监听下一个

由此，我们的调度器 `Scheduler` 可以同时做生产者-消费者任务，并且 handle 多个 socket 连接，实现了真正的异步、并发

## 7. 参考资料

> 本文来源于[David Beazley](https://www.youtube.com/user/dabeazllc) 的视频 [Build Your Own Async](https://www.youtube.com/watch?v=Y4Gt3Xjd7G8)，我在学习后，结合自己的理解写了本篇文章。我写的可能不够透彻，大家可以去原视频看看。
> 
> 源代码: [dabeaz/aproducer.py](https://gist.github.com/dabeaz/f86ded8d61206c757c5cd4dbb5109f74)


---

> 作者: [黄波](https://boh5.github.io)  
> URL: https://boh5.github.io/posts/blogs/basis/build-your-own-async/  

