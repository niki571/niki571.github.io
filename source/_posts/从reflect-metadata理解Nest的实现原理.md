---
title: 从 reflect metadata 理解 Nest 的实现原理
date: 2022-02-21 10:40:57
tags: 原理
categories: Node
---

# 依赖注入

Nest 是 Node.js 的服务端框架，它最出名的就是 IOC（inverse of control） 机制了，也就是不需要手动创建实例，框架会自动扫描需要加载的类，并创建他们的实例放到容器里，实例化时还会根据该类的构造器参数自动注入依赖。

它一般是这样用的：

比如入口 Module 里引入某个模块的 Module：

```javascript
import { Module } from "@nestjs/common";
import { CatsModule } from "./cats/cats.module";

@Module({
  imports: [CatsModule],
})
export class AppModule {}
```

然后这个模块的 Module 里会声明 Controller 和 Service：

```javascript
import { Module } from "@nestjs/common";
import { CatsController } from "./cats.controller";
import { CatsService } from "./cats.service";

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

Controller 里就是声明 url 对应的处理逻辑：

```javascript
import { Body, Controller, Get, Param, Post } from '@nestjs/common';
import { CatsService } from './cats.service';
import { CreateCatDto } from './dto/create-cat.dto';

@Controller('cats')
export class CatsController {
constructor(private readonly catsService: CatsService) {}

@Post()
async create(@Body() createCatDto: CreateCatDto) {
this.catsService.create(createCatDto);
}

@Get()
async findAll(): Promise<Cat[]> {
return this.catsService.findAll();
}
}
```

这个 CatsController 的构造器声明了对 CatsService 的依赖：

然后 CatsService 里就可以去操作数据库进行增删改查了。这里简单实现一下：

```javascript
import { Injectable } from '@nestjs/common';
import { Cat } from './interfaces/cat.interface';

@Injectable()
export class CatsService {
private readonly cats: Cat[] = [];

create(cat: Cat) {
this.cats.push(cat);
}

findAll(): Cat[] {
return this.cats;
}
}

```

之后在入口处调用 create 把整个 nest 应用跑起来：

```javascript
import { NestFactory } from "@nestjs/core";
import { AppModule } from "./app.module";

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  await app.listen(3000);
}
bootstrap();
```

然后浏览器访问下我们写的那个 controller 对应的 url，打个断点：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/86d0535710ad48348206520ee2cb89ee_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

你会发现 controller 的实例已经创建了，而且 service 也给注入了。这就是依赖注入的含义。

这种机制就叫做 IOC（控制反转），也叫依赖注入，好处是显而易见的，就是只需要声明依赖关系，不需要自己创建对象，框架会扫描声明然后自动创建并注入依赖。

Java 里最流行的 Spring 框架就是 IOC 的实现，而 Nest 也是这样一个实现了 IOC 机制的 Node.js 的后端框架。

不知道大家有没有感觉很神奇，只是通过装饰器声明了一下，然后启动 Nest 应用，这时候对象就给创建好了，依赖也给注入了。

那它是怎么实现的呢？

大家如果就这样去思考它的实现原理，还真不一定能想出来，因为缺少了一些前置知识。也就是实现 Nest 最核心的一些 api： Reflect 的 metadata 的 api。

# Reflect Metadata

有的同学会说，Reflect 的 api 我很熟呀，就是操作对象的属性、方法、构造器的一些 api：

比如 Reflect.get 是获取对象属性值

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/b5a7ba0e6dc94d0fb842cafc25ac4dd2_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

Reflect.set 是设置对象属性值

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/69c386cd587d4b958212a5ee1ee09a96_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

Reflect.has 是判断对象属性是否存在

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/783e8067a805459d9ec0d56292f529a4_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

Reflect.apply 是调用某个方法，传入对象和参数

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/7708d7a8c01a4fb6ae10b51b98442921_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

Reflect.construct 是用构造器创建对象实例，传入构造器参数

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/ed7bde270882465cb9ffdc624bbb4e61_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

这些 api 在 MDN 文档里可以查到，因为它们都已经是 es 标准了，也被很多浏览器实现了。

但是实现 Nest 用到的 api 还没有进入标准，还在草案阶段，也就是 metadata 的 api：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/a9cf799a18794e0a903170b6bf76fa21_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

它有这些 api：

```javascript
Reflect.defineMetadata(metadataKey, metadataValue, target);

Reflect.defineMetadata(metadataKey, metadataValue, target, propertyKey);

let result = Reflect.getMetadata(metadataKey, target);

let result = Reflect.getMetadata(metadataKey, target, propertyKey);
```

`Reflect.defineMetadata`和`Reflect.getMetadata`分别用于设置和获取某个类的元数据，如果最后传入了属性名，还可以单独为某个属性设置元数据。

那元数据存在哪呢？

存在类或者对象上呀，如果给类或者类的静态属性添加元数据，那就保存在类上，如果给实例属性添加元数据，那就保存在对象上，用类似`[[metadata]]`的`key`来存的。

这有啥用呢？

看上面的 api 确实看不出啥来，但它也支持装饰器的方式使用：

```javascript
@Reflect.metadata(metadataKey, metadataValue)
class C {
  @Reflect.metadata(metadataKey, metadataValue)
  method() {}
}
```

`Reflect.metadata`装饰器当然也可以再封装一层：

```javascript
function Type(type) {
  return Reflect.metadata("design:type", type);
}
function ParamTypes(...types) {
  return Reflect.metadata("design:paramtypes", types);
}
function ReturnType(type) {
  return Reflect.metadata("design:returntype", type);
}

@ParamTypes(String, Number)
class Guang {
  constructor(text, i) {}

  @Type(String)
  get name() {
    return "text";
  }

  @Type(Function)
  @ParamTypes(Number, Number)
  @ReturnType(Number)
  add(x, y) {
    return x + y;
  }
}
```

然后我们就可以通过 Reflect metadata 的 api 或者这个类和对象的元数据了：

```javascript
let obj = new Guang("a", 1);

let paramTypes = Reflect.getMetadata("design:paramtypes", obj, "add");
// [Number, Number]
```

这里我们用 Reflect.getMetadata 的 api 取出了 add 方法的参数的类型。

看到这里，大家是否明白 nest 的原理了呢？

我们再看下 nest 的源码：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/f2bb578b9b624bf993aaedc250ec053d_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

上面就是`@Module`装饰器的实现，里面就调用了 Reflect.defineMetadata 来给这个类添加了一些元数据。
所以我们这样用的时候：

```javascript
import { Module } from "@nestjs/common";
import { CatsController } from "./cats.controller";
import { CatsService } from "./cats.service";

@Module({
  controllers: [CatsController],
  providers: [CatsService],
})
export class CatsModule {}
```

其实就是给 CatsModule 添加了 controllers 的元数据和 providers 的元数据。

后面创建 IOC 容器的时候就会取出这些元数据来处理：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/12e7eca6e54e4fa1867b16b83237135a_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/32ea9f7aa5374ce681053a4dcae06723_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

而且 @Controller 和 @Injectable 的装饰器也是这样实现的：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/bd7235bd63374965a7d55c5866471983_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/c82a1514a28749668ec631b7e565e466_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

看到这里，大家是否想明白 nest 的实现原理了呢？

其实就是通过装饰器给 class 或者对象添加元数据，然后初始化的时候取出这些元数据，进行依赖的分析，然后创建对应的实例对象就可以了。

所以说，nest 实现的核心就是 Reflect metadata 的 api。

当然，现在 metadata 的 api 还在草案阶段，需要使用 reflect-metadata 这个 polyfill 包才行。

其实还有一个疑问，依赖的扫描可以通过 metadata 数据，但是创建的对象需要知道构造器的参数，现在并没有添加这部分 metadata 数据呀：

比如这个 CatsController 依赖了 CatsService，但是并没有添加 metadata 呀：

```javascript
import { Body, Controller, Get, Param, Post } from '@nestjs/common';
import { CatsService } from './cats.service';
import { CreateCatDto } from './dto/create-cat.dto';

@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  @Post()
  async create(@Body() createCatDto: CreateCatDto) {
    this.catsService.create(createCatDto);
  }

  @Get()
  async findAll(): Promise<Cat[]> {
    return this.catsService.findAll();
  }
}
```

这就不得不提到 TypeScript 的优势了，TypeScript 支持编译时自动添加一些 metadata 数据：

比如这段代码：

```javascript
import "reflect-metadata";

class Guang {
  @Reflect.metadata("名字", "光光")
  public say(a: number): string {
    return '加油鸭';
  }
}
```

按理说我们只添加了一个元数据，生成的代码也确实是这样的：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/4c525cd38ea542bab80ef31a15719265_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

但是呢，ts 有一个编译选项叫做 emitDecoratorMetadata，开启它就会自动添加一些元数据。

开启之后再试一下：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/d14d5736bef144a9a6830c7626b15b9f_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

你会看到多了三个元数据：

`design:type`是`Function`，很明显，这个是描述装饰目标的元数据，这里装饰的是函数

`design:paramtypes`是`[Number]`，很容易理解，就是参数的类型

`design:returntype`是`String`，也很容易理解，就是返回值的类型

所以说，只要开启了这个编译选项，ts 生成的代码会自动添加一些元数据。

然后创建对象的时候就可以通过 design:paramtypes 来拿到构造器参数的类型了，那不就知道怎么注入依赖了么？

所以，nest 源码里你会看到这样的代码：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/af6a8ad0ce814857881fbf2a7c503e7c_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

就是获取构造器的参数类型的。这个常量就是我们上面说的那个：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/3c6b32199ab1443794f56bdac63a2a5b_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

这也是为什么 nest 会用 ts 来写，因为它很依赖这个 emitDecoratorMetadata 的编译选项。

你用 cli 生成的代码模版里也都默认开启了这个编译选项：

![](https://raw.githubusercontent.com/niki571/MyImageHost/main/a984d979388a42bbb77bcef2b3c4dc26_tplv-k3u1fbpfcp-zoom-in-crop-mark_4536_0_0_0.webp)

这就是 nest 的核心实现原理：**通过装饰器给 class 或者对象添加 metadata，并且开启 ts 的 emitDecoratorMetadata 来自动添加类型相关的 metadata，然后运行的时候通过这些元数据来实现依赖的扫描，对象的创建等等功能。**

# 总结

Nest 是 Node.js 的后端框架，他的核心就是 IOC 容器，也就是自动扫描依赖，创建实例对象并且自动依赖注入。

要搞懂它的实现原理，需要先学习`Reflect metadata`的 api：

这个是给类或者对象添加`metadata`的。可以通过`Reflect.metadata`给类或者对象添加元数据，之后用到这个类或者对象的时候，可以通过`Reflect.getMetadata`把它们取出来。

`Nest`的`Controller`、`Module`、`Service`等等所有的装饰器都是通过 `Reflect.meatdata`给类或对象添加元数据的，然后初始化的时候取出来做依赖的扫描，实例化后放到`IOC`容器里。

实例化对象还需要构造器参数的类型，这个开启 ts 的`emitDecoratorMetadata`的编译选项之后， ts 就会自动添加一些元数据，也就是`design:type`、`design:paramtypes`、`design:returntype`这三个，分别代表被装饰的目标的类型、参数的类型、返回值的类型。

当然，`reflect metadata`的 api 还在草案阶段，需要引入 refelect metadata 的包做 polyfill。

nest 的一系列装饰器就是给 class 和对象添加 metadata 的，然后依赖扫描和依赖注入的时候就把 metadata 取出来做一些处理。

理解了 metadata，nest 的实现原理就很容易搞懂了。

```

```
