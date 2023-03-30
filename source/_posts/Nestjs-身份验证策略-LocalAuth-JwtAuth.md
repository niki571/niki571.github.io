---
title: Nestjs 身份验证策略 LocalAuth + JwtAuth
date: 2022-03-30 16:19:08
tags: [Nestjs]
categories: Node
---

本文通过使用 LocalAuth 进行身份验证，身份确认后，使用 JwtAuth 颁发身份令牌以及验证身份令牌的有效性。

# LocalAuth

Passport 提供了一种名为 passport-local 的策略，它实现了使用`用户名/密码`的身份验证机制。

首先安装需要的包：

```shell
npm install --save @nestjs/passport passport passport-local
npm install --save-dev @types/passport-local
```

在 nest 项目中创建 `AuthModule` 和 `AuthService`：

```shell
nest g module auth
nest g service auth
```

首先创建本地身份验证策略。在 `auth` 文件夹下创建 `strategies` 文件夹，然后创建 `local.strategy.ts` 文件。添加如下代码：

```typescript
import { Strategy } from "passport-local";
import { PassportStrategy } from "@nestjs/passport";
import { Injectable, UnauthorizedException } from "@nestjs/common";
import { AuthService } from "../auth.service";

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    // 默认不传任何参数，用户名和密码的字段是 username 和 password
    super();
    // 还可以定制我们自己的用户名和密码字段，如下：
    // 如果还有其他需求，可查看 Strategy 类的定义和使用方法
    super({
      usernameField: "account",
      passwordField: "pass",
    });
  }
  // 要实现的验证函数，固定写法
  async validate(username: string, password: string): Promise<any> {
    // 这里就可以写我们的身份验证流程，检测用户名和密码
    // 比如：
    // if(username !== 'admin' || password !== 'admin'){
    // throw new UnauthorizedException();
    // }
    // return {username, password};

    // 为了保证我们的策略指责单一，推荐把具体的验证逻辑放到authService中，此处只调用
    const user = await this.authService.validateUser(username, password);
    if (!user) {
      throw new UnauthorizedException();
    }
    return user;
  }
}
```

在 `authService.ts` 中写入验证身份逻辑，代码如下：

```typescript
import { Injectable } from "@nestjs/common";

@Injectable()
export class AuthService {
  constructor() {}

  async validateUser(name: string, password: string): Promise<any> {
    if (name !== "admin") {
      // UnauthorizedException异常会返回HTTP状态码是401的响应
      // 可根据我们的需求定制返回信息
      throw new UnauthorizedException({
        code: 401,
        message: "用户不存在",
      });
    } else if (password !== "admin") {
      throw new UnauthorizedException({
        code: 401,
        message: "密码错误",
      });
    }
    return { name, password };
  }
}
```

最后，还需要做的就是修改 `authModule.ts`，应用我们的 `providers`，也就是 `authService` 和 `LocalStrategy`。如下：

```typescript
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { PassportModule } from "@nestjs/passport";
import { LocalStrategy } from "./strategies/local.strategy";

@Module({
  imports: [PassportModule],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

至此，我们的本地身份验证策略就创建好了。下一步就是在登陆接口上使用此策略。`@nestjs/passport` 提供了一种内置的守卫 `AuthGuard`，通过在接口上应用此守卫，就可以实现本地验证策略流程。登陆接口示例如下：

```typescript
import { Post, Controller, Request, UseGuards } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";

@Controller("user")
export class UserController {
  constructor() {}

  @UseGuards(AuthGuard("local"))
  @Post("login")
  login(@Request() req): Record<string, any> {
    // req的user属性，是在passport-local身份验证流程中由Passport填充的
    return { ...req.user };
  }
}
```

请求登陆接口返回如下：

```typescript
{ name: 'admin', password: 'admin' }
```

至此，我们完成了身份验证过程。但我们期望的返回值并不是一个用户对象，应该是一个加密后的身份令牌，后续使用这个令牌去请求其他功能，这样才能保证我们业务过程中的身份安全和数据安全。接下来就使用 `jwt` 来颁发令牌，继续向下：

#JwtAuth

安装 `jwt` 相关的包：

```shell
npm install --save @nestjs/jwt passport-jwt
npm install --save-dev @types/passport-jwt
```

首先，创建生成 jwt 令牌的方法，在 `authService.ts` 中添加 `getToken` 方法：

```typescript
import { Injectable } from "@nestjs/common";
import { JwtService } from "@nestjs/jwt";

@Injectable()
export class AuthService {
  // ...

  getToken(user) {
    // 这里定义加密令牌的payload，可根据自己的需求定义
    const payload = { ...user };
    return this.jwtService.sign(payload);
  }
}
```

为了使 `jwtService` 生效，需要更新 `AuthMoudle` 导入 jwt 相关依赖并配置 `jwtModule`，如下：

首先创建 `auth/constants.ts` 文件，保存常量配置：

```typescript
export const jwtConstants = {
  secret: "secretKey", // 加密密钥
  expiresIn: "24h", // 24 小时内过期
};
```

然后更新 `AuthMoudle`：

```typescript
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { PassportModule } from "@nestjs/passport";
import { LocalStrategy } from "./strategies/local.strategy";
import { JwtModule } from "@nestjs/jwt";
import { jwtConstants } from "./constants";

@Module({
  imports: [
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: jwtConstants.expiresIn },
    }),
  ],
  providers: [AuthService, LocalStrategy],
})
export class AuthModule {}
```

更新登陆接口，获取令牌并返回：

```typescript
import { Post, Controller, Request, UseGuards } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
import { AuthService } from "../auth/auth.service";

@Controller("user")
export class UserController {
  constructor(private authService: AuthService) {}

  @UseGuards(AuthGuard("local"))
  @Post("login")
  login(@Request() req): Record<string, any> {
    // 生成令牌
    let token = this.authService.getToken(req.user);
    return { token };
  }
}
```

到这里，就可以实现登陆接口成功后返回 jwt 令牌：

```typescript
{
  token: "eyJhbGciOiJIUz1I1NiIbsInR5cjCI6IkpXVCJ9";
}
```

当我们的接口携带 jwt 令牌访问，服务器该如何检测令牌的有效性呢？接下来，就需要定义 jwt 验证策略，用来检测我们的令牌。

在 `auth/strategies` 文件夹中创建 `jwt.strategy.ts`，如下：

```typescript
import { ExtractJwt, Strategy } from "passport-jwt";
import { PassportStrategy } from "@nestjs/passport";
import { Injectable } from "@nestjs/common";
import { jwtConstants } from "../constants";

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor() {
    super({
      // 如何提取令牌
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      // 是否忽略令牌过期
      ignoreExpiration: false,
      // 解析令牌的密钥
      secretOrKey: jwtConstants.secret,
    });
  }
  // 验证成功的毁掉函数
  async validate(payload: any) {
    return { ...payload };
  }
}
```

在 `AuthModule` 中应用 jwt 策略：

```typescript
import { Module } from "@nestjs/common";
import { AuthService } from "./auth.service";
import { PassportModule } from "@nestjs/passport";
import { LocalStrategy } from "./strategies/local.strategy";
import { JwtStrategy } from "./strategies/jwt.strategy";
import { JwtModule } from "@nestjs/jwt";
import { jwtConstants } from "./constants";

@Module({
  imports: [
    PassportModule,
    JwtModule.register({
      secret: jwtConstants.secret,
      signOptions: { expiresIn: jwtConstants.expiresIn },
    }),
  ],
  providers: [AuthService, LocalStrategy, JwtStrategy],
})
export class AuthModule {}
```

到这里，jwt 策略就配置好了。可以为需要验证令牌的接口配置守卫，应用 jwt 策略，举例如下：

```typescript
import { Post, Get, Controller, Request, UseGuards } from "@nestjs/common";
import { AuthGuard } from "@nestjs/passport";
import { AuthService } from "../auth/auth.service";

@Controller("user")
export class UserController {
  // ...

  @UseGuards(AuthGuard("jwt"))
  @Get("get")
  getUser(@Request() req): Record<string, any> {
    return req.user;
  }
}
```
