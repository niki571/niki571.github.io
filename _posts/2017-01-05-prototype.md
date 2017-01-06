---
layout: post
title:  "最容易懂的原型继承总结"
date:   2017-01-05 23:23:30 +0800
categories: [Frontend]
excerpt:
tags:
  - CN
  - Javascript
---

这一篇是写给我的好朋友的，帮她梳理一下原型继承的知识～都是一些我自己的理解，如有不正确的地方欢迎指正～

### ES5中的原型继承

·大写的都是构造函数，

·构造函数可以通过new Person得到无数个有相同属性和相同方法却又可以有各自属性方法的实例

·就像我们都是中国人（属性）都可以说中文（方法），但是我们又来自祖国的不同地方（自己的属性）说不同的方言（自己的方法）一样

·this指向new出来的实例，你或者我或者他她它

```
var Person = function() {
  this.canTalk = true;
};
```

·构造函数的prototype是指原型

·大家共用的方法一般写在构造函数的原型上

·给Person类写上了共用的greet方法

```
Person.prototype.greet = function() {
  if (this.canTalk) {
    console.log('Hi, I am ' + this.name);
  }
};
```

·Employee是Person的子类

·通过call先把Person这个类的属性全都拿过来

·通过this.xxx又可以给Employee子类加上新的属性

```
var Employee = function(name, title) {
  Person.call(this);
  this.name = name;
  this.title = title;
};
```

·下面是非常关键的一步

·把 *子类的方法* 和 *子类本身* binding在一起

·首先子类的方法肯定是继承于父类的 => 通过Object.create => 创造子类的原型，这个原型继承于父类原型

·然后就是上面说的把这个子类原型的构造函数指向子类本身，binding

```
Employee.prototype = Object.create(Person.prototype);
Employee.prototype.constructor = Employee;
```

·最后在子类原型上定义子类自己的方法

```
Employee.prototype.greet = function() {
  if (this.canTalk) {
    console.log('Hi, I am ' + this.name + ', the ' + this.title);
  }
};
```

###### 让我们执行一下

全部代码如下，可以复制到console里面

```

```



### get的另类写法

### ES6的写法

