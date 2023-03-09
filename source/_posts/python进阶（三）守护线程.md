---
title: python 进阶（三）守护线程
date: 2021-09-10 15:46:38
tags: [python]
categories: python
---

在创建新线程时，子线程会从其父线程继承其线程属性，主线程是普通的非守护线程，默认情况下，它所创建的任何线程都是非守护线程。本文将介绍 Python 中的另一类后台线程，`守护线程（Daemon Thread）`。

# 无法退出的主线程

我们经常需要创建线程来执行某项例行任务或提供某种特殊服务，常见的例子就是垃圾收集器。垃圾收集器是一种自动内存管理器，它在后台运行并尝试回收程序不再使用的垃圾内存。许多语言都将垃圾收集都作为其运行时环境的标准部分。

<!-- more -->

**在默认情况下，新线程通常会生成非守护线程或普通线程，如果新线程在运行，主线程将永远等待，无法正常退出。**下面是一个普通线程的例子，在程序的主要部分中，我们创建并启动一个名为 olivia 的新厨房清洁线程，然后主线程打印出 Barron 正在烹饪的消息，最后 Barron 完成烹饪工作。

```python
#!/usr/bin/env python3
""" Barron finishes cooking while Olivia cleans """

import threading
import time

def kitchen_cleaner():
while True:
print("Olivia cleaned the kitchen.")
time.sleep(1)

if **name** == '**main**':
olivia = threading.Thread(target=kitchen_cleaner) # olivia.daemon = True
olivia.start()

    print('Barron is cooking...')
    time.sleep(0.6)
    print('Barron is cooking...')
    time.sleep(0.6)
    print('Barron is cooking...')
    time.sleep(0.6)
    print('Barron is done!')
```

在下面控制台输出结果中，我们看到来自 Barron 和 Olivia 的工作消息都显示出来了，但是在主线程结束并打印出最终的 Barron 完成消息后，程序并没有退出，因为厨房清洁器线程仍然运行，它将继续永远运行下去。

```shell
$ python daemon_thread.py
Olivia cleaned the kitchen.
Barron is cooking...
Barron is cooking...
Olivia cleaned the kitchen.
Barron is cooking...
Barron is done!
Olivia cleaned the kitchen.
Olivia cleaned the kitchen.
Olivia cleaned the kitchen.
Olivia cleaned the kitchen.
Olivia cleaned the kitchen.
Olivia cleaned the kitchen.
Olivia cleaned the kitchen.
...
```

# 变成守护线程

在前面的代码中我们定义了一个名为 kitchen_cleaner 的函数，它代表了一个像垃圾收集这样的周期性后台任务。厨房清洁器使用无限循环连续打印消息，Olivia 每秒清理厨房一次，下面我们将 Olivia 设置为守护程序线程，移除上面代码中的注释部分。

```python
olivia.daemon = True
```

当再次运行该程序时，**主线程执行完毕并且没有任何非守护线程继续运行时，主线程可以正常终止退出了**。

```shell
$ python daemon_thread.py
Olivia cleaned the kitchen.
Barron is cooking...
Barron is cooking...
Olivia cleaned the kitchen.
Barron is cooking...
Barron is done!
```

# 结束语

这里需要注意几点：

1. 必须在启动之前将线程配置为守护程序或非守护程序，否则 Python 将引发运行时错误；
2. 最后守护程序线程不会像普通线程一样正常退出，当程序中的所有非守护程序线程都完成执行时，任何剩余的守护程序线程将在 Python 退出时被放弃，在设计守护线程时，需要确保在主线程退出时不会产生任何负面影响。
