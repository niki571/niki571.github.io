---
title: python 进阶（一）asyncio
date: 2021-09-08 19:40:35
tags: [python]
categories: python
---

python 中协程概念是从 3.4 版本增加的，但 3.4 版本采用是生成器实现，为了将协程和生成器的使用场景进行区分，使语义更加明确，在 python 3.5 中增加了`async`和`await`关键字，用于定义原生协程。

# asyncio 异步 I/O 库

python 中的 asyncio 库提供了管理事件、协程、任务和线程的方法，以及编写并发代码的原语，即`async`和`await`。

<!-- more -->

该模块的主要内容：

- `事件循环`：`event_loop`，管理所有的事件，是一个无限循环方法，在循环过程中追踪事件发生的顺序将它们放在队列中，空闲时则调用相应的事件处理者来处理这些事件；
- `协程`：`coroutine`，子程序的泛化概念，协程可以在执行期间暂停，等待外部的处理（I/O 操作）完成之后，再从暂停的地方继续运行，函数定义式使用`async`关键字，这样这个函数就不会立即执行，而是返回一个协程对象；
- `Future`和`Task`：`Future`对象表示尚未完成的计算，`Task`是`Future`的子类，包含了任务的各个状态，作用是在运行某个任务的同时可以并发的运行多个任务。

## 异步函数的定义

异步函数本质上依旧是函数，只是在执行过程中会将执行权交给其它协程，与普通函数定义的区别是在`def`关键字前增加`async`。

```python
# 异步函数
import asyncio

# 异步函数
async def func(x):
    print("异步函数")
    return x \*\* 2

ret = func(2)
print(ret)
```

运行代码输入如下内容：

```shell
sys:1: RuntimeWarning: coroutine 'func' was never awaited
<coroutine object func at 0x0000000002C8C248>
```

函数返回一个协程对象，如果想要函数得到执行，需要将其放到事件循环 `event_loop`中。

## 事件循环 event_loop

`event_loop`是`asyncio`模块的核心，它将异步函数注册到事件循环上。
过程实现方式为：由`loop`在适当的时候调用协程，这里使用的方式名为`asyncio.get_event_loop()`，然后由`run_until_complete(协程对象)`将协程注册到事件循环中，并启动事件循环。

```python
import asyncio

# 异步函数
async def func(x):
    print("异步函数")
    return x \*\* 2

# 协程对象，该对象不能直接运行
coroutine1 = func(2)

# 事件循环对象
loop = asyncio.get_event_loop()

# 将协程对象加入到事件循环中，并执行
ret = loop.run_until_complete(coroutine1)
print(ret)
```

首先在 python 3.7 之前的版本中使用异步函数是安装上述流程：

1. 先通过`asyncio.get_event_loop()`获取事件循环`loop`对象；
2. 然后通过不同的策略调用`loop.run_until_complete()`或者`loop.run_forever()`执行异步函数。

在 python 3.7 之后的版本，直接使用`asyncio.run()`即可，该函数总是会创建一个新的事件循环并在结束时进行关闭。

最新的官方文档都采用的是`run`方法。

```python
import asyncio

async def main():
    print('hello')
    await asyncio.sleep(1)
    print('world')

asyncio.run(main())
```

接下来在查看一个完整的案例，并且结合`await`关键字。

```python
import asyncio
import time

# 异步函数 1
async def task1(x):
    print("任务 1")
    await asyncio.sleep(2)
    print("恢复任务 1")
    return x

# 异步函数 2
async def task2(x):
    print("任务 2")
    await asyncio.sleep(1)
    print("恢复任务 2")
    return x

async def main():
    start_time = time.perf_counter()
    ret_1 = await task1(1)
    ret_2 = await task2(2)
    print("任务 1 返回的值是", ret_1)
    print("任务 2 返回的值是", ret_2)
    print("运行时间", time.perf_counter() - start_time)

if __name__ == '__main__': # 创建一个事件循环
    loop = asyncio.get_event_loop() # 将协程对象加入到事件循环中，并执行
    loop.run_until_complete(main())
```

代码输出如下所示：

```shell
任务 1
恢复任务 1
任务 2
恢复任务 2
任务 1 返回的值是 1
任务 2 返回的值是 2
运行时间 2.99929154
```

上述代码创建了 3 个协程，其中`task1`和`task2`都放在了协程函数`main`中，I/O 操作通过`asyncio.sleep(1)`进行模拟，整个函数运行时间为 2.9999 秒，接近 3 秒，依旧是串行进行，如果希望修改为并发执行，将代码按照下述进行修改。

```python
import asyncio
import time

# 异步函数 1
async def task1(x):
    print("任务 1")
    await asyncio.sleep(2)
    print("恢复任务 1")
    return x

# 异步函数 2
async def task2(x):
    print("任务 2")
    await asyncio.sleep(1)
    print("恢复任务 2")
    return x

async def main():
    start_time = time.perf_counter()
    ret_1,ret_2 = await asyncio.gather(task1(1),task2(2))

    print("任务1 返回的值是", ret_1)
    print("任务2 返回的值是", ret_2)
    print("运行时间", time.perf_counter() - start_time)

if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
```

上述代码最大的变化是将`task1`和`task2`放到了`asyncio.gather()`中运行，此时代码输出时间明显变短。

```shell
任务 1
任务 2
恢复任务 2 # 任务 2 由于等待时间短，先返回。
恢复任务 1
任务 1 返回的值是 1
任务 2 返回的值是 2
运行时间 2.0005669480000003
```

`asyncio.gather()`可以更换为`asyncio.wait()`，修改代码如下所示：

```python
import asyncio
import time

# 异步函数 1
async def task1(x):
    print("任务 1")
    await asyncio.sleep(2)
    print("恢复任务 1")
    return x

# 异步函数 2
async def task2(x):
    print("任务 2")
    await asyncio.sleep(1)
    print("恢复任务 2")
    return x

async def main():
    start_time = time.perf_counter()
    done, pending = await asyncio.wait([task1(1), task2(2)])
    print(done)
    print(pending)

    print("运行时间", time.perf_counter() - start_time)

if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
```

`asyncio.wait()`返回一个元组，其中包含一个已经完成的任务集合，一个未完成任务的集合。

**`gather`和`wait`的区别**

- `gather`：需要所有任务都执行结束，如果任意一个协程函数崩溃了，都会抛异常，不会返回结果；
- `wait`：可以定义函数返回的时机，可以设置为`FIRST_COMPLETED`（第一个结束的）,`FIRST_EXCEPTION`（第一个出现异常的）,`ALL_COMPLETED`（全部执行完，默认的）。

```python
done,pending = await asyncio.wait([task1(1),task2(2)],return_when=asyncio.tasks.FIRST_EXCEPTION)
```

## 创建 task

由于协程对象不能直接运行，在注册到事件循环时，是`run_until_complete`方法将其包装成一个`task`对象。该对象是对`coroutine`对象的进一步封装，它比`coroutine`对象多了运行状态，例如`pending`，`running`，`finished`，可以利用这些状态获取协程对象的执行情况。

下面显示的将`coroutine`对象封装成`task`对象，在上述代码基础上进行修改。

```python
import asyncio
import time

# 异步函数 1
async def task1(x):
    print("任务 1")
    await asyncio.sleep(2)
    print("恢复任务 1")
    return x

# 异步函数 2
async def task2(x):
    print("任务 2")
    await asyncio.sleep(1)
    print("恢复任务 2")
    return x

async def main():
    start_time = time.perf_counter() # 封装 task 对象
    coroutine1 = task1(1)
    task_1 = loop.create_task(coroutine1)
    coroutine2 = task2(2)
    task_2 = loop.create_task(coroutine2)
    ret_1, ret_2 = await asyncio.gather(task_1, task_2)

    print("任务1 返回的值是", ret_1)
    print("任务2 返回的值是", ret_2)
    print("运行时间", time.perf_counter() - start_time)

if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
```

由于`task`对象是`future`对象的子类对象，所以上述代码也可以按照下述内容修改：

```python
# task_2 = loop.create_task(coroutine2)
task_2 = asyncio.ensure_future(coroutine2)
```

下面将`task`对象的各个状态进行打印输出。

```python
import asyncio
import time

# 异步函数 1
async def task1(x):
    print("任务 1")
    await asyncio.sleep(2)
    print("恢复任务 1")
    return x

# 异步函数 2
async def task2(x):
    print("任务 2")
    await asyncio.sleep(1)
    print("恢复任务 2")
    return x

async def main():
    start_time = time.perf_counter() # 封装 task 对象
    coroutine1 = task1(1)
    task_1 = loop.create_task(coroutine1)
    coroutine2 = task2(2) # task_2 = loop.create_task(coroutine2)
    task_2 = asyncio.ensure_future(coroutine2) # 进入 pending 状态
    print(task_1)
    print(task_2)

    # 获取任务的完成状态
    print(task_1.done(), task_2.done())
    # 执行任务
    await task_1
    await task_2
    # 再次获取完成状态
    print(task_1.done(), task_2.done())

    # 获取返回结果
    print(task_1.result())
    print(task_2.result())

    print("运行时间", time.perf_counter() - start_time)

if __name__ == '__main__':
    loop = asyncio.get_event_loop()
    loop.run_until_complete(main())
```

`await task_1`表示的是执行该协程，执行结束之后，`task.done()`返回`True`，`task.result()`获取返回值。

## 回调返回值

当协程执行完毕，需要获取其返回值，刚才已经演示了一种办法，使用`task.result()`方法获取，但是该方法仅当协程运行完毕时，才能获取结果，如果协程没有运行完毕，`result()`方法会返回`asyncio.InvalidStateError`（无效状态错误）。

一般编码都采用第二种方案，通过`add_done_callback()`方法绑定回调。

```python
import asyncio
import requests

async def request_html():
    url = 'https://www.csdn.net'
    res = requests.get(url)
    return res.status_code

def callback(task):
    print('回调：', task.result())

loop = asyncio.get_event_loop()

coroutine = request_html()
task = loop.create_task(coroutine)

# 绑定回调

task.add_done_callback(callback)
print(task)
print("*"*100)

loop.run_until_complete(task)
print(task)
```

上述代码当`coroutine`执行完毕时，会调用`callback`函数。

如果回调函数需要多个参数，请使用`functools`模块中的偏函数（`partial`）方法

## 循环事件关闭

建议每次编码结束之后，都调用循环事件对象`close()`方法，彻底清理`loop`对象。

# 本节课爬虫项目

本节课要采集的站点由于全部都是 coser 图片，所以地址在代码中查看即可。完整代码如下所示：

```python
import threading
import asyncio
import time
import requests
import lxml
from bs4 import BeautifulSoup

async def get(url):
    return requests.get(url)

async def get_html(url):
    print("准备抓取：", url)
    res = await get(url)
    return res.text

async def save_img(img_url): # thumbMid_5ae3e05fd3945 将小图替换为大图
    img_url = img_url.replace('thumb','thumbMid')
    img_url = "http://mycoser.com/" + img_url
    print("图片下载中：", img_url)
    res = await get(img_url)
    if res is not None:
    with open(f'./imgs/{time.time()}.jpg', 'wb') as f:
    f.write(res.content)
    return img_url,"ok"

async def main(url*list): # 创建 5 个任务
    tasks = [asyncio.ensure_future(get_html(url_list[*])) for _ in range(len(url_list))]

    dones, pending = await asyncio.wait(tasks)
    for task in dones:
        html = task.result()
        soup = BeautifulSoup(html, 'lxml')
        divimg_tags = soup.find_all(attrs={'class': 'workimage'})

        for div in divimg_tags:
            ret = await save_img(div.a.img["data-original"])
            print(ret)

if __name__ == '__main__':
    urls = [f"http://mycoser.com/picture/lists/p/{page}" for page in range(1, 17)]
    totle_page = len(urls) // 5 if len(urls) % 5 == 0 else len(urls) // 5 + 1 # 对 urls 列表进行切片，方便采集
    for page in range(0, totle_page):
        start_page = 0 if page == 0 else page * 5
        end_page = (page + 1) * 5

    # 循环事件对象
    loop = asyncio.get_event_loop()

    loop.run_until_complete(main(urls[start_page:end_page]))
```

**代码说明**上述代码中第一个要注意的是`await`关键字后面只能跟如下内容：

- 原生的协程对象；
- 一个包含`await`方法的对象返回的一个迭代器。

所以上述代码`get_html`函数中嵌套了一个协程`get`。主函数`main`里面为了运算方便，直接对 urls 进行了切片，然后通过循环进行运行。

当然上述代码的最后两行，可以直接修改为：

```python
# 循环事件对象
# loop = asyncio.get_event_loop()
#
# loop.run_until_complete(main(urls[start_page:end_page]))
asyncio.run(main(urls[start_page:end_page]))
```

轻松获取一堆高清图片。

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/721b7f428bf345d581fd1725456e3bbc_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.awebp)

# 写在后面

协程掌握了，python 爬虫之路就开启了。
