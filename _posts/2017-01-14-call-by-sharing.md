---
layout: post
title:  "javascript函数传参是按值传递还是按引用传递?"
date:   2017-01-14 15:47:30 +0800
categories: [Frontend]
excerpt:
tags:
  - CN
  - Javascript
---

最近在研究exports和module.exports，然后引发了我的思考。

### 函数传参

```
var num = 10,
    name = "AAA",
    obj1 = {
      value: "aaa"
    },
    obj2 = {
     value: "bbb"
    },
    obj3 = obj2;
 
function change(num, name, obj1, obj2) {
    num = num * 10;
    name = "BBB";
    obj1 = obj2;
    obj2.value = "ccc";
}
 
change(num, name, obj1, obj2);
 
console.log(num);  // 10
console.log(name); // "AAA"
console.log(obj1.value); //"aaa"
console.log(obj2.value); //"ccc"
console.log(obj3.value); //"ccc"
```

>* Number类型、String类型作为参数是按值传递，函数内部操作不影响外部作用域的值。
>* Object类型如果在函数内部被赋新值，函数内部操作不影响外部作用域的值。
>* Object类型如果在函数内部属性被赋新值，该object就是按引用传递，函数内部操作会影响外部作用域的值。

对比：

```
var num = 10,
    name = "AAA",
    obj1 = {
      value: "aaa"
    },
    obj2 = {
     value: "bbb"
    },
    obj3 = obj2;
 
function change() {
    num = num * 10;
    name = "BBB";
    obj1 = obj2;
    obj2.value = "ccc";
}
 
change();
 
console.log(num);  // 100
console.log(name); // "BBB"
console.log(obj1.value); //"ccc"
console.log(obj2.value); //"ccc"
console.log(obj3.value); //"ccc"
```

### ES6 有默认参数的函数传参

```
var x = 1;
function foo(x, y = function() { x = 2; }) {
  var x = 3;
  y();
  console.log(x);
}

foo() // 3
x // 1
```

>* 有默认参数的函数，函数外是全局作用域，函数参数是独立作用域，函数内部是局部作用域。

```
var x = 1;
function foo(x, y = function() { x = 2; }) {
  x = 3;
  y();
  console.log(x);
}

foo() // 2
x // 1
```

>* 去掉`var x = 3`的`var`，函数内部变量x指向函数参数作用域。

### exports和module.exports

在nodejs中。module是一个对象，每个模块中都有一个module对象，module是当前模块的一个引用。module.exports对象是Module系统创建的，而exports可以看作是对module.exports对象的一个引用。在模块中require另一个模块时，以module.exports的值为准，因为有的情况下，module.exports和exports它们的值是不同的。module.exports和exports的关系可以表示成这样：

```
// module.exports和exports相同的情况
var m = {};        // 表示 module
var e = m.e = {};  // e 表示 exports， m.e 表示 module.exports

m.e.a = 5;
e.b = 6;

console.log(m.e);  // Object { a: 5, b: 6 }
console.log(e);    // Object { a: 5, b: 6 }
```

```
// module.exports和exports不同的情况
var m = {};        // 表示 module
var e = m.e = {};  // e 表示 exports， m.e 表示 module.exports

m.e = { c: 9 };    // m.e（module.exports）引用的对象被改了
e.d = 10;

console.log(m.e);  // Object { c: 9 }
console.log(e);    // Object { d: 10 }
```

###### 参考资料

[Is JavaScript a pass-by-reference or pass-by-value language?](https://github.com/simongong/js-stackoverflow-highest-votes/blob/master/questions21-30/parameter-passed-by-value-or-reference.md)

[JS是按值传递还是按引用传递?](http://bosn.me/js/js-call-by-sharing/)

[javascript传递参数如果是object的话，是按值传递还是按引用传递？](https://www.zhihu.com/question/27114726)