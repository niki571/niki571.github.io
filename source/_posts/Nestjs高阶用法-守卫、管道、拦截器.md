---
title: Nestjs高阶用法-守卫、管道、拦截器
date: 2023-05-25 16:08:36
tags: [Nestjs]
categories: Node
---

# 前言

最近在写 Nestjs 后端项目，crud 写多了，发现关于守卫、管道、拦截器这些用的不多的东西反而生疏了，现在总结一下。

本篇主要实现以下功能：

- 使用`guards`和`decorators`实现数据校验核查
- 通过`interceptors`和`decorators`实现敏感数据录入
- 自定义`pipes`实现数据转化

# 守卫 Guards

在 **请求到达业务逻辑前** 会经过 `guard`，这样在接口前可以做统一处理。

例如：检查登陆态、检查权限...

需要在业务逻辑前 `统一检查` 的信息，都可以抽象成守卫。

在真实场景中，大多数的后台管理端会用 `JWT` 实现接口鉴权。`NestJs` 也提供了对应的解决方案。

由于较长且原理相通，本篇暂时用校验 `user` 字段做演示。

## 新建守卫

新建 `user.guard.ts` 文件

```typescript
// src/common/guards/user.guard.ts
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from "@nestjs/common";
import { Observable } from "rxjs";

@Injectable()
export class UserGuard implements CanActivate {
  canActivate(
    context: ExecutionContext
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.body.user;

    if (user) {
      return true;
    }

    throw new UnauthorizedException("need user field");
  }
}
```

## 单个接口使用守卫

单个接口使用需要用 `@UseGuards` 作为引用。再将定义的 `UserGuard` 作为入参。

在 `student.controller.ts` 中使用

```typescript
import { UseGuards /** ... **/ } from "@nestjs/common";
import { UserGuard } from "../common/guards/user.guard";
// ...

@Controller("students")
export class StudentsController {
  constructor(private readonly studentsService: StudentsService) {}

  @UseGuards(UserGuard)
  @Post("who-are-you")
  whoAreYouPost(@Body() student: StudentDto) {
    return this.studentsService.ImStudent(student.name);
  }
  // ...
}
```

这样当访问 `who-are-you` 和 `who-is-request` 就起作用了

```bash
// ❌ 不使用 user
curl -X POST http://127.0.0.1:3000/students/who-are-you -H 'Content-Type: application/json' -d '{"name": "gdccwxx"}'
// => {"statusCode":401,"message":"need user to distinct","error":"Unauthorized"}%

// ✅ 使用 user
// curl -X POST http://127.0.0.1:3000/students/who-are-you -H 'Content-Type: application/json' -d '{"user": "gdccwxx", "name": "gdccwxx"}'
// => Im student gdccwxx%
```

## 全局使用

全局使用仅需在 `app.module.ts` 的 `providers` 中引入。这样就对全局生效了

```typescript
import { APP_GUARD } from "@nestjs/core";
import { UserGuard } from "./common/guards/user.guard";
// ...

@Module({
  controllers: [AppController],
  providers: [
    {
      provide: APP_GUARD,
      useClass: UserGuard,
    },
    AppService,
  ],
  // ...
})
export class AppModule {}
```

这时再访问 `get` 请求 `who-are-you` 和 `post` 请求 `who-is-request`

```bash
// ❌ get who-are-you
http://localhost:3000/students/who-are-you?name=gdccwxx
// => {
// statusCode: 401,
// message: "need user field",
// error: "Unauthorized"
// }

// ✅ post
curl -X POST http://127.0.0.1:3000/students/who-is-request -H 'Content-Type: application/json' -d '{"user": "gdccwxx"}'
// => gdccwxx%
```

## 自定义装饰器过滤

总有些接口我们不需要有 `user` 字段，这时自定义 `decorator` 就出马了。

基本原理是：在接口前设置 `MetaData`, 在服务启动时把 `MetaData` 写入内存，这样在请求过来时判断有无 `MetaData` 标签。有则通过，无则校验。

顺便也将 `get` 请求类型过滤掉。

```typescript
// common/decorators.ts
export const NoUser = () => SetMetadata("no-user", true);
```

`user.guard.ts` 改造

```typescript
// user.guard.ts
import { Reflector } from "@nestjs/core";
// ..

@Injectable()
export class UserGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(
    context: ExecutionContext
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    const user = request.body.user;

    if (request.method !== "POST") {
      return true;
    }

    const noCheck = this.reflector.get<string[]>(
      "no-user",
      context.getHandler()
    );

    if (noCheck) {
      return true;
    }

    if (user) {
      return true;
    }

    throw new UnauthorizedException("need user field");
  }
}
```

`NoUser` 使用

```typescript
// students.controller.ts
import { User, NoUser } from "../common/decorators";
// ..

@Controller("students")
export class StudentsController {
  // ...
  @NoUser()
  @Post("who-are-you")
  whoAreYouPost(@Body() student: StudentDto) {
    return this.studentsService.ImStudent(student.name);
  }
}
```

再调用时，就不会再校验了。

```bash
// ✅
curl -X POST http://127.0.0.1:3000/students/who-are-you -H 'Content-Type: application/json' -d '{"name": "gdccwxx"}'
// => Im student gdccwxx%
```

这样就实现了全局守卫，但是部分接口不需要守卫的情况。

特别适用于登录态的校验，只有登陆接口不需要登录态，其他接口都需要登陆态或鉴权。

# 拦截器 Interceptors

拦截器工作在 `请求前` 和 `响应后`。它的原理和 `decorator` 类似，不同的是能做全局级别。

它的应用场景也非常广，例如：接口请求参数和请求结果的数据保存、设计模式中的 adapter 模式等...

我们来用它实现敏感信息的数据保存。

它的原理和 `guards` 类似, 通过 `decorator` 加载到内存，知道哪些接口需要敏感操作记录，然后在调用接口时将 入参和结果存入。

涉及到数据库操作，因此需要新增模块和数据库连接。

## 新建敏感权限模块

新建敏感权限模块，包括 `controller`、`module` 和 `service`

```bash
nest g controller sensitive
nest g module sensitive
nest g service sensitive
```

## 创建 entity 文件

新建 `sensitive.entity.ts`。

这里会用到 `transformer`, 原因是 `mysql` 底层并没有 `Object` 类型。需要通过 `JS` 把它存成 `string` 格式，在读取时用 `object` 格式。这样代码就不需要感知是啥类型了。

```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
} from "typeorm";
import { SensitiveType } from "../constants";

// to 写入数据库
// from 从数据库读取
const dataTransform = {
  to: (value: any) => JSON.stringify(value || {}),
  from: (value: any) => JSON.parse(value),
};

@Entity()
export class Sensitive {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ type: "enum", enum: SensitiveType })
  type: string;

  @Column({ type: "varchar" })
  pathname: string;

  @Column({ type: "text", transformer: dataTransform })
  parameters: any;

  @Column({ type: "text", transformer: dataTransform })
  results: any;

  @CreateDateColumn()
  createDate: Date;
}
```

## 引用数据库

和之前介绍数据库一样，在 `sensitive.module.ts` 中引入数据库

```typescript
import { Module } from "@nestjs/common";
import { SensitiveController } from "./sensitive.controller";
import { SensitiveService } from "./sensitive.service";
import { Sensitive } from "./entities/sensitive.entity";
import { TypeOrmModule } from "@nestjs/typeorm";

@Module({
  controllers: [SensitiveController],
  imports: [TypeOrmModule.forFeature([Sensitive])],
  providers: [Sensitive, SensitiveService],
  exports: [SensitiveService],
})
export class SensitiveModule {}
```

## service 核心逻辑

敏感操作比较简单，service 仅需实现新增和查询。

先定义敏感操作类型

```typescript
// src/sensitive/constants.ts
export enum SensitiveType {
  Modify = "Modify",
  Set = "Set",
  Create = "Create",
  Delete = "Delete",
}
```

在修改 `service`，引入 `db` 操作

```typescript
// src/sensitive/sensitive.service.ts
import { Injectable } from "@nestjs/common";
import { Sensitive } from "./entities/sensitive.entity";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { SensitiveType } from "./constants";

@Injectable()
export class SensitiveService {
  constructor(
    @InjectRepository(Sensitive)
    private readonly sensitiveRepository: Repository<Sensitive>
  ) {}

  async setSensitive(
    type: SensitiveType,
    pathname: string,
    parameters: any,
    results: any
  ) {
    return await this.sensitiveRepository
      .save({
        type,
        pathname,
        parameters,
        results,
      })
      .catch((e) => e);
  }

  async getSensitive(type: SensitiveType) {
    return await this.sensitiveRepository.find({
      where: {
        type,
      },
    });
  }
}
```

## controller 修改

`controller` 比较简单，只需要简单的查询即可。敏感信息写入则是通过 `decorator + interceptor` 来实现

```typescript
// src/sensitive/sensitive.controller.ts
import { Controller, Get, Query } from "@nestjs/common";
import { SensitiveService } from "./sensitive.service";
import { SensitiveType } from "./constants";

@Controller("sensitive")
export class SensitiveController {
  constructor(private readonly sensitiveService: SensitiveService) {}

  @Get("/get-by-type")
  getSensitive(@Query("type") type: SensitiveType) {
    return this.sensitiveService.getSensitive(type);
  }
}
```

## 新增装饰器

装饰器的用场来了，只需要告诉某个接口需要敏感操作记录，并指定类型即可。

```typescript
// src/common/decorators
import { SetMetadata } from "@nestjs/common";
import { SensitiveType } from "../sensitive/constants";

export const SensitiveOperation = (type: SensitiveType) =>
  SetMetadata("sensitive-operation", type);
// ...
```

通过传参的方式，定义敏感操作的类型。在数据库中可以分类，通过索引的方式查找修改入参和结果。

## 拦截器

重点来了！！

和守卫权限校验类似，通过 `reflector` 取出内存中的 `sensitive-operation` 类型。

```typescript
// src/common/interceptors/sensitive.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from "@nestjs/common";
import { Reflector } from "@nestjs/core";
import { SensitiveService } from "../../sensitive/sensitive.service";
import { SensitiveType } from "../../sensitive/constants";
import { Observable } from "rxjs";
import { tap } from "rxjs/operators";

@Injectable()
export class SensitiveInterceptor implements NestInterceptor {
  constructor(
    private reflector: Reflector,
    private sensitiveService: SensitiveService
  ) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();

    const type = this.reflector.get<SensitiveType | undefined>(
      "sensitive-operation",
      context.getHandler()
    );

    if (!type) {
      return next.handle();
    }

    return next
      .handle()
      .pipe(
        tap((data) =>
          this.sensitiveService.setSensitive(
            type,
            request.url,
            request.body,
            data
          )
        )
      );
  }
}
```

并在 `app.module.ts` 中引入全局。

```typescript
// app.module.ts
import { APP_GUARD, APP_INTERCEPTOR } from "@nestjs/core";
import { SensitiveInterceptor } from "./common/interceptors/sensitive.interceptor";
// ...

@Module({
  providers: [
    {
      provide: APP_INTERCEPTOR,
      useClass: SensitiveInterceptor,
    },
    // ...
  ],
  // ...
})
export class AppModule {}
```

## 其他模块引用

`student` 模块引入

```typescript
// src/students/students.controller.ts
import { SensitiveOperation } from "../common/decorators";
import { SensitiveType } from "../sensitive/constants";
// ...

@Controller("students")
export class StudentsController {
  constructor(private readonly studentsService: StudentsService) {}

  @SensitiveOperation(SensitiveType.Set)
  @Post("set-student-name")
  setStudentName(@User() user: string) {
    return this.studentsService.setStudent(user);
  }
  // ...
}
```

仅需要在接口前引入 `@SensitiveOperation(SensitiveType.Set)` 即可！是不是非常优美。

再来调用下！

```bash
/ ✅ 使用命令行调用
curl -X POST http://127.0.0.1:3000/students/set-student-name -H 'Content-Type: application/json' -d '{"user": "gdccwxx1"}'
// => {"name":"gdccwxx1","id":3,"updateDate":"2021-09-17T05:48:41.685Z","createDate":"2021-09-17T05:48:41.685Z"}%

// ✅ 打开浏览器
http://localhost:3000/sensitive/get-by-type?type=set
// => [{
// id: 1,
// type: "Set",
// pathname: "/students/set-student-name",
// parameters: { user: "gdccwxx1" },
// results: { name: "gdccwxx1", id: 3, updateDate: "2021-09-17T05:48:41.685Z", createDate: "2021-09-17T05:48:41.685Z" },
// createDate: "2021-09-17T05:48:41.719Z"
// }]
```

bingo！这样就达到了我们想要的目的！

在不影响原有业务逻辑的情况下，仅是在接口处做标识的简单调用。实现了 AOP 的调用方式。对老代码的改造和新业务的编写都十分有用。

# 管道 Pipes

`NestJs Pipes` 的概念和 `linux shell` 的概念非常相似，都是通过前者的输出再做一些事情。

它的应用场景也非常广，例如：数据转化，数据校验等...

对数据输入时的操作非常有用。对复杂数据校验，例如表单数据等十分有用。

我们没有复杂输入，我们来使用简单的数据转化，实现在名字前加上 🇨🇳

## 新建 Pipes

```typescript
// src/common/pipes/name.pipes.ts
import { PipeTransform, Injectable, ArgumentMetadata } from "@nestjs/common";

@Injectable()
export class TransformNamePipe implements PipeTransform {
  transform(name: string, metadata: ArgumentMetadata) {
    return `🇨🇳 ${name.trim()}`;
  }
}
```

和其他 `NestJs` 一样，都需要重载一边内置对象。`Pipes` 也需要重载 `PipeTransform`。

## 使用管道

在 `controller` 中使用 `pipes`。

```typescript
import { TransformNamePipe } from "../common/pipes/name.pipes";
// ...

@Controller("students")
export class StudentsController {
  constructor(private readonly studentsService: StudentsService) {}

  @Get("who-are-you")
  whoAreYou(@Query("name", TransformNamePipe) name: string) {
    return this.studentsService.ImStudent(name);
  }
  // ...
}
```

`query` 的第二个参数是 `pipes`, 也可以使用多个 `pipes` 对数据连续处理

## 调用接口

再浏览器访问

```bash
// ✅
http://localhost:3000/students/who-are-you?name=gdccwxx
// => Im student 🇨🇳 gdccwxx
```

这样就实现了简单版本的数据转换了！

# 总结

至此，`NestJs` 的入门篇章就结束了。

简单回顾下教程内容：

- 使用 `guard` 对参数进行校验（可扩展成登录态）
- 使用 `interceptor` 实现敏感数据落地
- 使用 `pipes` 实现数据格式化

在 `NestJs` 逐渐探索中发现，它不仅包括简单的数据服务，还支持 `GraphQL`、`SSE`、`Microservice` 等等，是综合性非常强的框架。
