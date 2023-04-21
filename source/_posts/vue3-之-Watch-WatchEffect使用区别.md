---
title: vue3 之 Watch WatchEffect 的使用和区别
date: 2022-08-12 16:09:19
tags: [Vue]
categories: Vue
---

曾经以为自己会用 `watch` 、 `watchEffect` 了，后来发现只是略懂皮毛。最近我就把 Vue3 的侦听器全面梳理了一下，分享给大家。看看有没有你不会的吧，一起学起来！

# Watch

## 基本用法

当我们需要在数据变化时执行一些“副作用”：如更改 DOM、执行异步操作，我们可以使用 `watch` 函数：

<!-- more -->

```typescript
<script setup>
import { ref, watch } from 'vue'

const question = ref('')
const answer = ref('This is answer. ;-)')

// 侦听一个 ref
watch(question, async (newQuestion, oldQuestion) => {
  answer.value = 'Thinking...'
  const res = await fetch('https://...')
  answer.value = (await res.json()).answer
})
</script>

<template>
  <input v-model="question" />
  <p>{{ answer }}</p>
</template>
```

`watch()` 一共可以接受三个参数，侦听数据源、回调函数和配置选项。

## 侦听数据源

`watch` 的第一个参数可以是不同形式的“数据源”，它可以是：

- 一个 ref
- 一个计算属性
- 一个 getter 函数（有返回值的函数）
- 一个响应式对象
- 以上类型的值组成的数组

```typescript
const x = ref(1)
const y = ref(1)
const doubleX = computed(() => x.value \* 2)
const obj = reactive({ count: 0 })

// 单个 ref
watch(x, (newValue) => {
console.log(`x is ${newValue}`)
})

// 计算属性
watch(doubleX, (newValue) => {
console.log(`doubleX is ${newValue}`)
})

// getter 函数
watch(
() => x.value + y.value,
(sum) => {
console.log(`sum of x + y is: ${sum}`)
}
)

// 响应式对象
watch(obj, (newValue, oldValue) => {
// 在嵌套的属性变更时触发
// 注意：`newValue` 此处和 `oldValue` 是相等的
// 因为它们是同一个对象！
})

// 以上类型的值组成的数组
watch([x, () => y.value], ([newX, newY]) => {
console.log(`x is ${newX} and y is ${newY}`)
})
```

注意，你不能直接侦听响应式对象的属性值，例如：

```typescript
const obj = reactive({ count: 0 });

// 错误，因为 watch() 得到的参数是一个 number
watch(obj.count, (count) => {
  console.log(`count is: ${count}`);
});
```

这里需要用一个返回该属性的 getter 函数：

```typescript
// 提供一个 getter 函数
watch(
  () => obj.count,
  (count) => {
    console.log(`count is: ${count}`);
  }
);
```

## 回调函数

`watch` 的第二个参数是数据发生变化时执行的回调函数。

这个回调函数接受三个参数：新值、旧值，以及一个用于清理副作用的回调函数。该回调函数会在副作用下一次执行前调用，可以用来清除无效的副作用，如等待中的异步请求：

```typescript
const id = ref(1);
const data = ref(null);

watch(id, async (newValue, oldValue, onCleanup) => {
  const { response, cancel } = doAsyncWork(id.value);
  // `cancel` 会在 `id` 更改时调用
  // 以便取消之前未完成的请求
  onCleanup(cancel);
  data.value = await response.json();
});
```

`watch`的返回值是一个用来停止该副作用的函数：

```typescript
const unwatch = watch(() => {});

// ...当该侦听器不再需要时
unwatch();
```

注意：使用同步语句创建的侦听器，会自动绑定到宿主组件实例上，并且会在宿主组件卸载时自动停止。使用异步回调创建一个侦听器，则不会绑定到当前组件上，你必须手动停止它，以防内存泄漏。如下面这个例子：

```typescript
<script setup>
import { watchEffect } from 'vue'

// 组件卸载会自动停止
watchEffect(() => {})

// 组件卸载不会停止！
setTimeout(() => {
  watchEffect(() => {})
}, 100)
</script>
```

## 配置选项

`watch` 的第三个参数是一个可选的对象，支持以下选项：

- `immediate`：在侦听器创建时立即触发回调。
- `deep`：深度遍历，以便在深层级变更时触发回调。
- `flush`：回调函数的触发时机。pre：默认，dom 更新前调用，post: dom 更新后调用，sync 同步调用。
- `onTrack / onTrigger`：用于调试的钩子。在依赖收集和回调函数触发时被调用。

## 深层侦听器

直接给 `watch()` 传入一个响应式对象，会默认创建一个深层侦听器 —— 所有嵌套的属性变更时都会被触发：

```typescript
const obj = reactive({ count: 0 });

watch(obj, (newValue, oldValue) => {
  // 在嵌套的属性变更时触发
  // 注意：`newValue` 此处和 `oldValue` 是相等的
  // 因为它们是同一个对象！
});

obj.count++;
```

相比之下，一个返回响应式对象的 getter 函数，只有在对象被替换时才会触发：

```typescript
const obj = reactive({
  someString: "hello",
  someObject: { count: 0 },
});

watch(
  () => obj.someObject,
  () => {
    // 仅当 obj.someObject 被替换时触发
  }
);
```

当然，你也可以显式地加上 `deep` 选项，强制转成深层侦听器：

```typescript
watch(
  () => obj.someObject,
  (newValue, oldValue) => {
    // `newValue` 此处和 `oldValue` 是相等的
    // 除非 obj.someObject 被整个替换了
    console.log("deep", newValue.count, oldValue.count);
  },
  { deep: true }
);

obj.someObject.count++; // deep 1 1
```

深层侦听一个响应式对象或数组，新值和旧值是相等的。为了解决这个问题，我们可以对值进行深拷贝。

```typescript
watch(
  () => _.cloneDeep(obj.someObject),
  (newValue, oldValue) => {
    // 此时 `newValue` 此处和 `oldValue` 是不相等的
    console.log("deep", newValue.count, oldValue.count);
  },
  { deep: true }
);

obj.someObject.count++; // deep 1 0
```

注意：深层侦听需要遍历所有嵌套的属性，当数据结构庞大时，开销很大。所以我们要谨慎使用，并且留意性能。

# watchEffect

`watch()` 是懒执行的：当数据源发生变化时，才会执行回调。但在某些场景中，我们希望在创建侦听器时，立即执行一遍回调。当然使用 `immediate` 选项也能实现：

```typescript
const url = ref("https://...");
const data = ref(null);

async function fetchData() {
  const response = await fetch(url.value);
  data.value = await response.json();
}

// 立即执行一次，再侦听 url 变化
watch(url, fetchData, { immediate: true });
```

可以看到 `watch` 用到了三个参数，我们可以用 `watchEffect` 来简化上面的代码。`watchEffect` 会立即执行一遍回调函数，如果这时函数产生了副作用，Vue 会自动追踪副作用的依赖关系，自动分析出侦听数据源。上面的例子可以重写为：

```typescript
const url = ref("https://...");
const data = ref(null);

// 一个参数就可以搞定
watchEffect(async () => {
  const response = await fetch(url.value);
  data.value = await response.json();
});
```

`watchEffect` 接受两个参数，第一个参数是数据发生变化时执行的回调函数，用法和 `watch` 一样。第二个参数是一个可选的对象，支持 `flush` 和 `onTrack / onTrigger` 选项，功能和 `watch` 相同。

注意：`watchEffect` 仅会在其同步执行期间，才追踪依赖。使用异步回调时，只有在第一个 `await` 之前访问到的依赖才会被追踪。

# watch vs. watchEffect

`watch` 和 `watchEffect` 的主要功能是相同的，都能响应式地执行回调函数。它们的区别是追踪响应式依赖的方式不同：

- `watch` 只追踪明确定义的数据源，不会追踪在回调中访问到的东西；默认情况下，只有在数据源发生改变时才会触发回调；watch 可以访问侦听数据的新值和旧值。
- `watchEffect` 会初始化执行一次，在副作用发生期间追踪依赖，自动分析出侦听数据源；`watchEffect`  无法访问侦听数据的新值和旧值。

简单一句话，`watch` 功能更加强大，而 `watchEffect` 在某些场景下更加简洁。
