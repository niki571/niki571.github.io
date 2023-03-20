---
title: 用 javascript 写一个队列
date: 2023-03-20 14:44:29
tags:
---

之前我们介绍过 python 的 Queue 队列，队列可以解决并发过程中生产和消费不同步的问题。具体可以翻看[前文](https://markdown.com.cn)。

今天我们会以 Yocto-queue 源码为例，看一下 javascript 如何写一个队列。

# Yocto-queue

Yocto-queue 是一种允许高效存储和检索数据的数据结构。它是一种队列类型，是一个元素集合，其中的项被添加到一端并从另一端移除。
它被设计用来操作数据量很大的数组，在你需要使用大量的 Array.push、Array.shift 操作时，Yocto-queue 有更好的性能表现。

# 源码分析

```javascript
class Node {
  value;
  next;

  constructor(value) {
    this.value = value;
  }
}

export default class Queue {
  #head;
  #tail;
  #size;

  constructor() {
    this.clear();
  }

  enqueue(value) {
    const node = new Node(value);

    if (this.#head) {
      this.#tail.next = node;
      this.#tail = node;
    } else {
      this.#head = node;
      this.#tail = node;
    }

    this.#size++;
  }

  dequeue() {
    const current = this.#head;
    if (!current) {
      return;
    }

    this.#head = this.#head.next;
    this.#size--;
    return current.value;
  }

  clear() {
    this.#head = undefined;
    this.#tail = undefined;
    this.#size = 0;
  }

  get size() {
    return this.#size;
  }

  *[Symbol.iterator]() {
    let current = this.#head;

    while (current) {
      yield current.value;
      current = current.next;
    }
  }
}
```

## 队列

队列是一种先进先出（FIFO）的数据结构，具有以下几个特点：

- 新元素总是添加到队列的末尾。
- 已经在队列中的元素保持原有的顺序不变。
- 任何时候，只能从队列的开头（顶部）删除元素。

## 入队

向队列中添加值。该方法需要一个值作为参数，它用来创建一个新的 Node 对象。

如果队列中已经有一个 head 和 tail 节点，新节点将会添加到队列末尾，通过将 tail 节点的 next 属性设置为新节点，并更新 tail 属性为新节点。

如果队列为空，新节点将成为 head 和 tail 节点。最后，队列的 size 属性会增加以反映新添加的节点。

## 出队

从队列中删除顶部节点的值，并将其返回。

它首先通过检查 head 属性是否为空来检查队列是否为空。如果队列为空，该方法返回 null。如果队列不为空，head 属性将更新为队列中的下一个节点，并且 size 属性减少以反映删除的节点。然后返回原 head 节点的值。

## 迭代器

允许在 for...of 循环中使用 yocto-queue。

使用 Symbol.iterator 符号来为队列定义一个自定义迭代器。迭代器首先将 current 变量设置为队列的 head 属性。然后进入一个循环，只要 current 不为 null 就继续循环。每次迭代，都会使用 yield 关键字产生 current 节点的 value 属性。然后 current 变量将更新为队列中的下一个节点，循环继续。这样 for...of 循环就可以遍历队列中的所有值。

# 总结

通过阅读`yocto-queue`的源码，学习到了队列的实现方式，以及迭代器的使用。数组 以及 队列两种数据结构在使用场景上的异同，数组是查询快，插入慢，队列是查询慢，插入快。
