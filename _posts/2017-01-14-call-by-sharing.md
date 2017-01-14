---
layout: post
title:  "javascript按值传递还是按引用传递?"
date:   2017-01-14 15:47:30 +0800
categories: [Frontend]
excerpt:
tags:
  - CN
  - Javascript
---

最近在研究exports和module.exports，然后引发了我的思考。

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

[Is JavaScript a pass-by-reference or pass-by-value language?](https://github.com/simongong/js-stackoverflow-highest-votes/blob/master/questions21-30/parameter-passed-by-value-or-reference.md)

[JS是按值传递还是按引用传递?](http://bosn.me/js/js-call-by-sharing/)