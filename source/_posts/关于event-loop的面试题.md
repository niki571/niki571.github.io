---
title: 关于 event loop 的面试题
date: 2020-09-30 17:57:07
tags: [面试, javascript]
categories: 前端
---

题目 1

```javascript
console.log("1");
setTimeout(function () {
  console.log("2");
  process.nextTick(function () {
    console.log("3");
  });
  new Promise(function (resolve) {
    console.log("4");
    resolve();
  }).then(function () {
    console.log("5");
  });
});

process.nextTick(function () {
  console.log("6");
});

new Promise(function (resolve) {
  console.log("7");
  resolve();
}).then(function () {
  console.log("8");
});

setTimeout(function () {
  console.log("9");
  process.nextTick(function () {
    console.log("10");
  });
  new Promise(function (resolve) {
    console.log("11");
    resolve();
  }).then(function () {
    console.log("12");
  });
});
```

> 结果 1 7 6 8 2 4 3 5 9 11 10 12

<!-- more -->

题目 2

```javascript
new Promise(function (resolve) {
  console.log("1"); // 宏任务一
  resolve();
}).then(function () {
  console.log("3"); // 宏任务一的微任务
});
setTimeout(function () {
  // 宏任务二
  console.log("4");
  setTimeout(function () {
    // 宏任务五
    console.log("7");
    new Promise(function (resolve) {
      console.log("8");
      resolve();
    }).then(function () {
      console.log("10");
      setTimeout(function () {
        // 宏任务七
        console.log("12");
      });
    });
    console.log("9");
  });
});
setTimeout(function () {
  // 宏任务三
  console.log("5");
});
setTimeout(function () {
  // 宏任务四
  console.log("6");
  setTimeout(function () {
    // 宏任务六
    console.log("11");
  });
});
console.log("2"); // 宏任务一
```

> 结果 1 2 3 4 5 6 7 8 9 10 11 12

题目 3

```javascript
async function async1() {
  console.log("1");
  await new Promise((resolve) => {
    console.log("2");
    resolve();
  }).then(() => {
    console.log("3");
  });
  console.log("4");
}
async1();
```

> 结果 1 2 3 4

题目 4

```javascript
setTimeout(function () {
  console.log("6");
}, 0);
console.log("1");
async function async1() {
  console.log("2");
  await async2();
  console.log("5");
}
async function async2() {
  console.log("3");
}
async1();
console.log("4");
```

> 结果 1 2 3 4 5 6

题目 5

```javascript
console.log("1");
async function async1() {
  console.log("2");
  await "await的结果";
  console.log("5");
}

async1();
console.log("3");

new Promise(function (resolve) {
  console.log("4");
  resolve();
}).then(function () {
  console.log("6");
});
```

> 结果 1 2 3 4 5 6

题目 6

```javascript
async function async1() {
  console.log("2");
  await async2();
  console.log("7");
}

async function async2() {
  console.log("3");
}

setTimeout(function () {
  console.log("8");
}, 0);

console.log("1");
async1();

new Promise(function (resolve) {
  console.log("4");
  resolve();
}).then(function () {
  console.log("6");
});
console.log("5");
```

> 结果 1 2 3 4 5 7 6 8

题目 7

```javascript
setTimeout(function () {
  console.log("9");
}, 0);
console.log("1");
async function async1() {
  console.log("2");
  await async2();
  console.log("8");
}
async function async2() {
  return new Promise(function (resolve) {
    console.log("3");
    resolve();
  }).then(function () {
    console.log("6");
  });
}
async1();

new Promise(function (resolve) {
  console.log("4");
  resolve();
}).then(function () {
  console.log("7");
});
console.log("5");
```

> 结果 1 2 3 4 5 6 7 8 9

题目 8

```javascript
async function async1() {
  console.log("2");
  const data = await async2();
  console.log(data);
  console.log("8");
}

async function async2() {
  return new Promise(function (resolve) {
    console.log("3");
    resolve("await的结果");
  }).then(function (data) {
    console.log("6");
    return data;
  });
}
console.log("1");

setTimeout(function () {
  console.log("9");
}, 0);

async1();

new Promise(function (resolve) {
  console.log("4");
  resolve();
}).then(function () {
  console.log("7");
});
console.log("5");
```

> 结果 1 2 3 4 5 6 7 await 的结果 8 9

题目 9

```javascript
setTimeout(function () {
  console.log("8");
}, 0);

async function async1() {
  console.log("1");
  const data = await async2();
  console.log("6");
  return data;
}

async function async2() {
  return new Promise((resolve) => {
    console.log("2");
    resolve("async2的结果");
  }).then((data) => {
    console.log("4");
    return data;
  });
}

async1().then((data) => {
  console.log("7");
  console.log(data);
});

new Promise(function (resolve) {
  console.log("3");
  resolve();
}).then(function () {
  console.log("5");
});
```

> 结果 1 2 3 4 5 6 7 async2 的结果 8

题目 10

```javascript
async function test() {
  console.log("test start");
  await undefined;
  console.log("await 1");
  await new Promise((r) => {
    console.log("promise in async");
    r();
  });
  console.log("await 2");
}

test();
new Promise((r) => {
  console.log("promise");
  r();
})
  .then(() => {
    console.log(1);
  })
  .then(() => {
    console.log(2);
  })
  .then(() => {
    console.log(3);
  })
  .then(() => {
    console.log(4);
  });
```

> 结果 test start -> promise -> await 1 -> promise in async -> 1 -> await 2 -> 2 -> 3 -> 4
