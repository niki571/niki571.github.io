---
title: python 进阶（二）queue
date: 2021-09-09 13:39:40
tags: [python]
categories: python
---

# 生产者消费者模型

在并发编程中，比如爬虫，有的线程负责爬取数据，有的线程负责对爬取到的数据做处理（清洗、分类和入库）。假如他们是直接交互的，那么当二者的速度不匹配时势必出现等待现象，这也就产生了资源的浪费。

抽象是一种很重要的通用能力，而生产者消费者模型是前人将一系列同类型的具体的问题抽象出来的一个一致的最佳解决方案。

该模型有三个重要角色，`容器`，`生产者`和`消费者`，顾名思义，`生产者`就是负责生产数据或任务的，`消费者`就是负责消费数据或者任务的（下文统称为任务），`容器`是二者进行通讯的媒介。

<!-- more -->

在该模型中，生产者和消费者不在直接进行通讯，而是通过引入一个第三者容器（通常都是用阻塞队列）来达到解耦的目的。这样生产者不必在因为消费者速度过慢而等待，直接将任务放入容器即可，消费者也不必因生产者生产速度过慢而等待，直接从容器中获取任务，以此达到了资源的最大利用。

使用该模型可以解决并发编程中的绝大部分并发问题。

# 简易版

我们先写一个单生产者和单消费者的简易版生产者消费者模型。

```python
import threading
import time
import queue

def consume(thread_name, q):
    while True:
        time.sleep(2)
        product = q.get()
        print("%s consume %s" % (thread_name, product))

def produce(thread_name, q):
    for i in range(3):
        product = 'product-' + str(i)
        q.put(product)
        print("%s produce %s" % (thread_name, product))
        time.sleep(1)

q = queue.Queue()
p = threading.Thread(target=produce, args=("producer",q))
c = threading.Thread(target=consume, args=("consumer",q))

p.start()
c.start()

p.join()

# 输出如下
producer produce product-0
producer produce product-1
consumer consume product-0
producer produce product-2
consumer consume product-1
consumer consume product-2
...
```

以上就是最简单的生产者消费者模型了，生产者生产三个任务供消费者消费。但是上面的写法有个问题，就是生产者将任务生产完毕之后就和主线程一起退出了，但是消费者将所有的任务消费完之后还没停止，一直处于阻塞状态。

那可不可以将`while True`的判断改为`while not q.empty()`呢，肯定是不行的。因为`empty()`返回`False`，不保证后续调用的`get()`不被阻塞。同时，如果用`empty()`函数来做判断的话，那么就要保证消费者线程开启之时生产者一定至少生产了一个任务，否则消费者线程就会因条件不满足直接退出程序；同时如果生产者生产速度比较慢，一旦消费者将任务消费完且下次判断时还没有新的任务入队，那么消费者线程也会因条件不满足直接退出程序。自此以后，生产者生产的任务就永远不会被消费了。

那我们可以做一个约定，当生产者生产完任务之后，放入一个标志，类似于`q.put(None)`，一旦消费者接收到为 None 的任务时就意味着结束，直接退出程序即可。这种做法在上面的程序中是没有问题的，唯一的缺点就是有 N 个消费者线程就需要放入 N 个 None 标志，这对于多消费者类型的程序显然是很不友好的。

# 最佳实践

我们可以结合队列的内置函数`task_done()`和`join()`来达到我们的目的。

`join()`函数是阻塞的。当消费者通过`get()`从队列获取一项任务并处理完成之后，需要调用且只可以调用一次`task_done()`，该方法会给队列发送一个信号，`join()`函数则在监听这个信号。可以简单理解为队列内部维护了一个计数器，该计数器标识未完成的任务数，每当添加任务时，计数器会增加，调用`task_done()`时计数器则会减少，直到队列为空。而`join()`就是在监听队列是否为空，一旦条件满足则结束阻塞状态。

```python
import threading
import time
import queue

def consume(thread_name, q):
    while True:
        time.sleep(2)
        product = q.get()
        print("%s consume %s" % (thread_name, product))
        q.task_done()

def produce(thread_name, q):
    for i in range(3):
        product = 'product-' + str(i)
        q.put(product)
        print("%s produce %s" % (thread_name, product))
        time.sleep(1)
        q.join()

q = queue.Queue()
p = threading.Thread(target=produce, args=("producer",q))
c = threading.Thread(target=consume, args=("consumer",q))
c1 = threading.Thread(target=consume, args=("consumer-1",q))

c.setDaemon(True)
c1.setDaemon(True)
p.start()
c.start()
c1.start()

p.join()

# 输出如下
producer produce product-0
producer produce product-1
consumer-1 consume product-0
consumer consume product-1
producer produce product-2
consumer consume product-2
```

上述示例中，我们将消费者线程设置为守护线程，这样当主线程结束时消费者线程也会一并结束。然后主线程最后一句`p.join()`又表示主线程必须等待生产者线程结束后才可以结束。

再细看生产者线程的主函数`produce()`，该函数中出现了我们上面说过的`q.join()`函数。而`task_done`则是在消费者线程的主函数中调用的。故当生产者线程生产完所有任务后就会被阻塞，只有当消费者线程处理完所有任务后生产者才会阻塞结束。随着生产者线程的结束，主线程也一并结束，守护线程消费者线程也一并结束，自此所有线程均安全退出。

# Queue 总结

本章节介绍了队列的高级应用，从简易版的示例到最佳实践，介绍了生产者消费者模型的基本用法，在该模型中，队列扮演了非常重要的角色，起到了解耦的目的。

本模型有固定的步骤，其中最重要的就是通过`task_done()`和`join()`来互相通信。`task_done()`仅仅用来通知队列消费者已完成一个任务，至于任务是什么它毫不关心，它只关心队列中未完成的任务数量。注意：`task_done()`不可以在`put()`之前调用，否则会引发 ValueError: task_done() called too many times。同时在处理完任务后只可以调用一次该函数，否则队列将不能准确计算未完成任务数量。
