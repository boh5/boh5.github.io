# Python 高级编程（十三）asyncio


## 1. 事件循环

- 并发编程3要素：事件循环 + 回调（协程中为驱动生成器） + IO 多路复用
- asyncio 是 python 用于解决异步 IO 编程的一整套解决方案
- python 的异步编程框架：tornado、gevent、twisted(scrapy、django channels)
- 同步阻塞的函数不要放在协程里，否则在 loop 中会阻塞
- `loop.create_task()` 等于 `asyncio.ensure_future()`，返回 Task 对象，可用于获取协程返回值，在已运行的 loop 中还可以使用 `asyncio.create_task()`
- Task 对象上可设置回调 `task.add_done_callback(callback_func)`，其中 callback_func 有一个 future 参数 `def callback_func(future)`，或使用 `functools.partial` 来传递更多参数
- python 3.10 中 `asyncio.get_event_loop()` warning deprecated，使用 `asyncio.new_event_loop()` 或 `asyncio.get_running_loop()` 代替
- wait vs gather：gather 更加 high-level；gather 返回一个 Future 对象，可用于分组，可以 `cancle()`

## 2. task 取消和子协程调用原理

- loop 会被放到 future 中（因此可以在 future 中stop loop，这也是 `loop.run_until_complete()` 的实现方式：给 future 加上 done_callback，然后 `loop.run_forever()`），future 也会被放到 loop 中

子协程调用原理：[18.5.3.1.3. Example: Chain coroutines](https://docs.python.org/3.6/library/asyncio-task.html#example-chain-coroutines)


## 3. loop 的 call_soon、call_at、call_later、call_soon_threadsafe

- `loop.call_soon()`：安排在下一次事件循环的迭代中调用 callback
- `loop.call_later()`：安排 callback 在给定的 delay 秒（可以是 int 或者 float）后被调用。
- `loop.call_at()`：安排 callback 在给定的绝对时间戳的 时间 （一个 int 或者 float）被调用，使用与 loop.time() 同样的时间参考。
- `loop.call_soon_threadsafe()`：asyncio 可以在多线程下运行，是一个异步io解决方案(协程、线程、进程)，不光是解决协程问题

## 4. ThreadPoolExecutor + asyncio

协程中是不能有阻塞 IO的，但如果非得在协程中集成阻塞 IO，那么可以用多线程解决

```python
loop = asyncio.get_event_loop()
executor = concurrent.futures.ThreadPoolExecutor(1)
tasks = []
for i in range(20):
    url = 'http://baidu.com'
    task = loop.run_in_executor(executor, get_url, url)  # 返回 future 对象
    tasks.append(task)
loop.run_until_complete(asyncio.wait(tasks))
```

## 4. asyncio 模拟 http 请求

```python
async def get_url(url):
    url = urllib.parse.urlparse(url)
    host = url.netloc
    path = url.path if url.path != '' else '/'

    reader, writer = await asyncio.open_connection(host, 80)

    writer.write('GET {} HTTP/1.1\r\nHost:{}\r\nConnection:Close\r\n\r\n'.format(path, host).encode('utf8'))

    all_lines = []
    async for raw_line in reader:  # StreamReader 实现了 async def __anext__()，因此可以用 async for
        line = raw_line.decode()
        all_lines.append(line)

    html = '\n'.join(all_lines)
    return html


async def main():
    tasks = []
    for i in range(20):
        tasks.append(asyncio.ensure_future(get_url('http://baidu.com')))

    for task in asyncio.as_completed(tasks):  # 获取结果
        result = await task
        print(result)


if __name__ == '__main__':
    start = time.time()
    loop = asyncio.new_event_loop()
    loop.run_until_complete(main())
    print(time.time() - start)
```

## 6. future 和 task

- future 是一个结果容器
- task 是 future 的子类，是协程和 future 之间的桥梁，为了 asyncio 和线程池、进程池接口一致


## 7. asyncio 的同步和通信

- `asyncio.Lock()`
- `asyncio.Queue()`


---

> 作者: [黄波](https://boh5.com)  
> URL: https://boh5.com/posts/notes/languages/python/imooc/13-asyncio/  

