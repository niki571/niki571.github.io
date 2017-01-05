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

这一篇是写给我的好朋友的，帮她梳理一下原型继承的知识～

### ES5中的原型继承

```
//大写的都是构造函数，
//构造函数可以通过new Person得到无数个有相同属性和相同方法却又可以有自己各自属性方法的实例
//就像我们都是中国人（属性）都可以说话（方法），但是我们又有不同的身高体重（属性）不同的做饭技巧（方法）一样
//this指向new出来的实例，你或者我或者他她它
var Person = function() {
  this.canTalk = true;
};
//构造函数的prototype是指原型
//大家共用的方法一般写在构造函数的原型上
Person.prototype.greet = function() {
  if (this.canTalk) {
    console.log('Hi, I am ' + this.name);
  }
};
//雇员继承于人
//通过call先把Person这个类的属性全都拿过来
//通过this又可以加上雇员这个类的新属性
var Employee = function(name, title) {
  Person.call(this);
  this.name = name;
  this.title = title;
};
//这是非常关键的一步
//相当于把雇员这个子类的**方法**安在雇员这个类上，binding的既视感
//首先子类的方法肯定是继承于父类的=>Object.create=>创造子类的原型，这个原型包含父类原型
//然后就是上面说的把这个子类原型的构造函数指向子类本身，binding
Employee.prototype = Object.create(Person.prototype);
Employee.prototype.constructor = Employee;
//最后在子类上定义方法
Employee.prototype.greet = function() {
  if (this.canTalk) {
    console.log('Hi, I am ' + this.name + ', the ' + this.title);
  }
};
```

### get的另类写法

### ES6的写法

