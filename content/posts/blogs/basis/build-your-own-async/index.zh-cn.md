---
title: 深入理解并发，从零开始构建你自己的 Async
date: 2022-08-28T15:00:00+08:00
lastmod: 2022-08-28T16:35:00+08:00
categories:
  - Blog
tags:
  - Python
  - Python3
  - Async
  - EventLoop
draft: true
---

> https://www.youtube.com/watch?v=Y4Gt3Xjd7G8
> https://python.plainenglish.io/build-your-own-event-loop-from-scratch-in-python-da77ef1e3c39

两种方式：
1. 基于回调
2. 基于协程

基本调度，回调：

1. 并行方式
2. 多线程实现并发
3. 假如没有多线程我们该怎么做
4. 实现 scheduler 及 call_soon, run，time.sleep() 会 block 住，原方法中 for 要改为 call_soon()
5. sleep 以参数传入 call_later
6. 改用 heapq 实现 self.sleeping 为优先级队列，替代排序
7. heapq 如果两个 function 有相同的 deadline，那么排序会出问题，因为不能对 function 排序，添加一个 sequence number 来解决

生产者消费者问题，回调：

1. 以线程实现
2. 不用线程该怎么做呢，改造之前的 scheduler
3. 实现自己的 AsyncQueue
4. for 改为 call_later，由 scheduler 来调度
5. 添加 AsyncQueue().close() 方法，并实现类似 Future 的 Result 对象
6. 这样的回调实现，不好，很容易进入回调地狱，很难阅读和理解
7. 不能在其中使用循环和 Blocking 的代码

基本调度，生成器：

1. 实现 run 和 new_task 的 scheduler
2. 我们的函数，除了多了个 yield 就像普通函数一样
3. 有没有可能不用 yield 呢，毕竟看起来有点滥用，不太好看
4. 构建一个 Awaitable 对象，用 await Awaitable() 代替 yield，但必须在 async func 中，这样方法就变成了 corotine(协程)
5. 这样就不能用 next 调度了，要用 coro.send(None)
6. 这样只是隐藏了 yield，但其实在 Awaitbale 中还是有 yield，只是用户看不到实现的细节
7. 但是如何处理 sleep 呢，现在还是会 block 代码
8. 要在 scheduler 中实现一个 async sleep 方法:
  - 把 current coroutine 放入 sleeping 队列
  - 把 self.current 置为 None，让 run 方法调度下一个 coroutine
  - run 方法中如果 ready 队列没东西，就去 sleeping 里找，时间不到，就 time.sleep 等待。注意 coro.send(None)，是让当前方法执行到下一个 yield
  - asyncio 中的 sleep 方法也是类似的原理


生产者消费者，生成器：

1. sleep 改为 sched.sleep()，使用 async 和 await
2. 实现 AsyncQueue，put 方法简单
3. get 方法得为 coroutine，
   - queue 中没有数据的时候，要把 coroutine 从 shced 中拿掉，避免反复执行
   - 拿掉的 coroutine 放入 queue 的等待队列，queue put 方法调用时，要从等待队列拿出一个 coroutine 放入 sched 的 ready 队列，以供调用
4. put 方法也改为 coroutine 使其和 get 对称
5. 实现 close 方法，不再需要像基于回调的实现那样处理异常，因为对于 get 方法，如果队列中没东西再去做是否 close 的判断
6. get 方法改 while，以便 close 后，get 方法可以识别到


结合回调和生成器，asyncio 的实际方式

1. 实现 Task 类封装 coroutine，通过 new_task 方法把 Task 放入 ready 队列，执行的时候，把从 Task 传入的 coroutine 进行调度，并放到回调的 ready 队列，
2. 再次从 ready 队列取出 Task 时，如果执行完成，会抛出 StopIteration 异常，结束调度
3. sleep 方法，执行 call_later 方法把 current coroutine 放入 sleeping 队列

实现异步I/O socket 为例




