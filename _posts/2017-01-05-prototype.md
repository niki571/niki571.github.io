---
layout: post
title:  "最容易懂的ES5ES6原型继承总结"
date:   2017-01-05 23:23:30 +0800
categories: [Frontend]
excerpt:
tags:
  - CN
  - Javascript
---

这一篇是写给我的好朋友的，帮她梳理一下原型继承的知识～都是一些我自己的理解，如有不正确的地方欢迎指正～

### ES5中的原型继承

>* 大写的都是构造函数
>* 构造函数可以通过new Person得到无数个有相同属性和相同方法却又可以有各自属性方法的实例
>* 就像我们都是中国人（属性）都可以说中文（方法），但是我们又来自祖国的不同地方（自己的属性）说不同的方言（自己的方法）一样
>* this指向new出来的实例，你或者我或者他她它

```
var Person = function() {
  this.canTalk = true;
};
```

>* 构造函数的prototype是指原型
>* 大家共用的方法写在构造函数的原型上
>* 给Person类写上了共用的greet方法

```
Person.prototype.greet = function() {
  if (this.canTalk) {
    console.log('Hi, I am ' + this.name);
  }
};
```

>* Employee是Person的子类
>* 通过call先把Person这个类的属性全都拿过来
>* 通过this.xxx又可以给Employee子类加上新的属性

```
var Employee = function(name, title) {
  Person.call(this);
  this.name = name;
  this.title = title;
};
```

>* 把父类的方法继承到子类上 => 通过Object.create => 创造子类的原型，这个原型继承于父类原型
>* 在下一节会介绍假如没有这句话会怎样

```
Employee.prototype = Object.create(Person.prototype);
```

>* 再把子类原型的构造函数指向子类本身
>* 在下一节会介绍假如没有这句话会怎样

```
Employee.prototype.constructor = Employee;
```

>* 最后在子类原型上定义子类自己的方法

```
Employee.prototype.bigGreet = function() {
  if (this.canTalk) {
    console.log('Hi, I am ' + this.name + ', the ' + this.title);
  }
};
```

>* new一个实例

```
var bob = new Employee('Bob', 'Builder');
```

###### 让我们执行一下

全部代码如下，可以复制到console里面

```
var Person = function() {
  this.canTalk = true;
};

Person.prototype.greet = function() {
  if (this.canTalk) {
    console.log('Hi, I am ' + this.name);
  }
};

var Employee = function(name, title) {
  Person.call(this);
  this.name = name;
  this.title = title;
};
//去掉这一句试试
Employee.prototype = Object.create(Person.prototype); 
//保留上一句，去掉这一句再试试
Employee.prototype.constructor = Employee;

Employee.prototype.bigGreet = function() {
  if (this.canTalk) {
    console.log('Hi, I am ' + this.name + ', the ' + this.title);
  }
};
var bob = new Employee('Bob', 'Builder');
```

>分别执行`bob.greet()`和`bob.bigGreet()`, Bob都正确的和我们打招呼了～
>
>![characters](../../assets/images/prototype/right_output.png)

>让我们去掉`Employee.prototype = Object.create(Person.prototype);`，再执行一次：
>
>![characters](../../assets/images/prototype/wrong_output1.png)
>
>报错了，父类Person的方法没有继承过来

>这次去掉`Employee.prototype.constructor = Employee;`,再执行一次：
>
>![characters](../../assets/images/prototype/wrong_output2.png)
>
>Bob同样正确的和我们打招呼了，那为什么还要加上这句话呢？验证一下：
>
>![characters](../../assets/images/prototype/constructor.png)
>
>发现子类原型构造函数指向了父类而不是子类本身，这是因为`Employee.prototype = Object.create(Person.prototype);`造成的，这当然不是我们想要的，所以这句话就是为了把构造函数重新指向子类自己


### get的另类写法

### ES6的写法

