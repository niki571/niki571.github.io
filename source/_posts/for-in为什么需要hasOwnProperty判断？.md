---
title: for...in为什么需要hasOwnProperty判断？
date: 2020-03-11 20:32:10
tags: [javascript, 面试]
categories: 前端
---

我们经常使用 for...in 循环对象输出键和值

如：

```javascript
const obj = { a: 1, b: 2 };
for (let key in obj) {
  console.log(key); // => a b
  console.log(obj[key]); // => 1 2
}
```

但在某些编辑器（如 vscode）中输入 for in，会自动生成如下代码：

```javascript
for (const key in object) {
  if (Object.hasOwnProperty.call(object, key)) {
    const element = object[key];
    // todo ..
  }
}
```

每次都会想这句是不是多余的呢？

<!-- more -->

`(Object.hasOwnProperty.call(object, key)`

引用 MDN 解释，`for...in`语句以任意顺序迭代一个对象的除`Symbol`以外的可枚举属性，包括继承的可枚举属性。

重点：**包括继承的可枚举属性**

下面看一个例子：

```javascript
function School() {
  this.school = "清北";
}

function Person() {
  this.name = "小明";
  this.age = "18";
}

Person.prototype = new School();

var p = new Person();

// 我们要打印 p 上面的属性
for (let key in p) {
  console.log(`${key} = ${p[key]}`);
}
// 输出
// name = 小明
// age = 18
// school = 南翔
```

`Person.prototype = new School()`

Person 的原型上继承了 School，输出了`school = 南翔`，这里不明白的话建议看看你红宝书继承那一章，这里简单使用，不做解释！

我们的目标是输出 p 自己的属性，明显这不符合如期，所以 hasOwnProperty 派上用场了

引用 MDN 解释：

`hasOwnProperty()`方法会返回一个布尔值，指示对象自身属性中是否具有指定的属性（也就是，是否有指定的键）

```javascript
// ..省略
for (let key in p) {
  if (Object.hasOwnProperty.call(p.key)) {
    console.log(`${key} = ${p[key]}`);
  }
}
// 输出
// name = 小明
// age = 18
```

当然，小伙伴工作时候一般遍历`let obj = {}`这种对象，其实可以省略，但是如果我们开发一些工具库的时候还是严谨点好。

延伸问题：

`Object.hasOwnProperty.call(p. key)`

hasOwnProperty 是 Object 上的方法，所有对象默认会继承该方法，为什么不直接

`p.hasOwnProperty(key)`

而使用

`Object.hasOwnProperty.call(object, key)`

我们试试：

```javascript
// ..省略
for (let key in p) {
  if (p.hasOwnProperty(key)) {
    console.log(`${key} = ${p[key]}`);
  }
}
// 输出
// name = 小明
// age = 18
```

好明显是符合如期的!

为啥要这么写呢，那肯定有它的原因！！！

假设我们把代码改一下：

```javascript
// ..其他省略
function Person() {
  this.name = "小明";
  this.age = "18";
  this.hasOwnProperty = function () {
    return false;
  };
}
for (let key in p) {
  if (p.hasOwnProperty(key)) {
    console.log(`${key} = ${p[key]}`);
  }
}
// 输出 ??? 啥也没有
```

好明显，我们定义 hasOwnProperty 把原来的方法覆盖了，虽然我们平时不会这样做，但还是那句话，如果我们开发工具或者库得时候，写代码还是得严谨点好！
