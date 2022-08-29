---
title: Python 高级编程（十二）协程
last_modified_at: 2022-08-10T22:14+08:00
toc: true
toc_sticky: true
excerpt_separator: <!--more-->
categories:
  - 学习笔记
  - python
tags:
  - python
  - python3
  - 协程
---

## 1. 并发、并行、同步、异步、阻塞、非阻塞

- 并发：**一个时间段内**在**同一个 cpu 上**有多个程序在运行，但**任意时刻**只有一个程序在运行
- 并行：**任意时刻**多个程序同时运行在**多个 cpu** 上
- 同步：代码调用 IO 操作时，**必须等待** IO 操作完成才返回
- 异步：代码调用 IO 操作时，**不必等待** IO 操作完成才返回
- 阻塞：调用函数时，当前线程被挂起
- 非阻塞：调用函数时，单曲线程不会被挂起，而是立即返回

<!--more-->

## 2. C10K 问题和 IO 多路复用

### 2.1 C10K问题

- 很难用线程解决，很难开启 10K 个线程

### 2.2 Linux 五种 IO 模型

[Linux IO 模型](https://dilless.github.io/%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0/redis/redis-principle-2-network/#1-linux-io-%E6%A8%A1%E5%9E%8B)


### 2.3 select + 回调 + 事件循环模拟 http 请求

```python
import selectors
import socket
import time
import urllib.parse

selector = selectors.DefaultSelector()  # 根据不同平台自动选择最好的 io 多路复用方式

urls = ['http://www.baidu.com'] * 20
stop = False


class Fetcher:
    def __init__(self):
        self.host = None
        self.path = None
        self.client = None
        self.cur_url = None
        self.data = b''

    def get_url(self, url):
        self.cur_url = url
        url = urllib.parse.urlparse(url)
        self.host = url.netloc
        self.path = url.path if url.path != '' else '/'

        # 建立 socket 连接
        self.client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.client.setblocking(False)

        try:
            self.client.connect((self.host, 80))
        except BlockingIOError as e:
            pass

        # 把 socket 注册到 selector 上
        selector.register(self.client.fileno(), selectors.EVENT_WRITE, self.connected)  # 回调模式，当这个 socket 上可写时，调用

    def connected(self, key):
        selector.unregister(key.fd)
        self.client.send(
            'GET {} HTTP/1.1\r\nHost:{}\r\nConnection:Close\r\n\r\n'.format(self.path, self.host).encode('utf8'))
        selector.register(self.client.fileno(), selectors.EVENT_READ, self.readable)

    def readable(self, key):  # 准备好一段读一段，该函数可能有多次 EVENT_READ 多次被调用
        d = self.client.recv(1024)
        if d:
            self.data += d
        else:
            selector.unregister(key.fd)

            data = self.data.decode('utf8')
            html = data.split('\r\n\r\n')[1]
            print(html)
            self.client.close()
            urls.remove(self.cur_url)
            if not urls:
                global stop
                stop = True


def loop():
    # 事件循环，不停的请求 socket 的状态，并调用对应的回调函数
    # twisted、gevent、asyncio 本质上来讲都是这种模式：回调 + 事件循环 + select\poll\epoll
    # 1. select 本身不支持 register 模式(回调)的
    # 2. socket 状态变化以后的回调是由我们的程序完成的
    while not stop:
        ready = selector.select()
        for key, mask in ready:
            call_back = key.data
            call_back(key)


def get_url(url):
    url = urllib.parse.urlparse(url)
    host = url.netloc
    path = url.path if url.path != '' else '/'

    client = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    client.setblocking(False)
    try:
        client.connect((url.netloc, 80))
    except BlockingIOError as e:
        pass

    while True:
        try:
            client.send('GET {} HTTP/1.1\r\nHost:{}\r\nConnection:Close\r\n\r\n'.format(path, host).encode('utf8'))
        except OSError as e:
            pass
        else:
            break

    data = b''
    while True:
        try:
            d = client.recv(1024)
        except BlockingIOError as e:
            continue
        if d:
            data += d
        else:
            break

    data = data.decode('utf8')
    html = data.split('\r\n\r\n')[1]
    print(html)
    client.close()


if __name__ == '__main__':
    # 异步
    start = time.time()
    for url in urls:
        fetcher = Fetcher()
        fetcher.get_url(url)
    loop()
    print('select:', time.time() - start)

    # 同步
    urls = ['http://www.baidu.com'] * 20
    start = time.time()
    for url in urls:
        get_url(url)
    print('for   :', time.time() - start)

```

### 2.4 回调之痛

问题：
- 异常不由主函数捕获，需要在 loop 中处理，难以处理
- 嵌套回调，层数多了难以理解和维护，如果某一层出异常，难以处理
- 变量在回调间共享难以维护

总结：
- 可读性差
- 共享状态管理困难
- 异常处理困难

## 3. 协程

### 3.1 C10M 问题

随互联网发展，C10K 都不够用了

### 3.2 协程

要实现线程内切换，需要可暂停的函数，并且可以在适当的时候恢复以继续执行

协程：可暂停的函数，可以向暂停的地方传入值

### 3.3 生成器高级特性

启动生成器的方法：
- `gen.send(None)`
- `next(gen)`

其他方法：
- `gen.close()`：关闭生成器
- `gen.throw()`：向上次暂停的地方传入异常

return 值：
- 运行到 return 语句会抛出 `StopIteration` 异常，`e.value` 就是返回值


### 3.4 yield from

`itertools.chan()` 可以连接多个 Iterable 对象

`yield from [sub-generator | iterable]` 会在调用方和子生成器之间建立一个双向通道：
- `next()` 会从子生成器 yield 出一个值
- `send()` 会发送到子生成器
- `throw()` 会发送到子生成器
- 子生成器的 return 值会返回到 yield from 所在行，赋值给左边（把子生成器抛出的 `StopIteration` 异常中的 `value` 复制给左边）。

### 3.5 async 和 await

python 为了语义明确，就引入了 async 和 await 关键词用于定义原生协程


