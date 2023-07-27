---
title: NestJS 入门（四）Redis 和注销
date: 2023-07-27 13:48:41
index_img: https://img.bald3r.wang/img/20230712153658.png
categories:
  - Node.js
  - NestJS
tags:
  - Node.js
  - NestJS
---

# NestJS 入门（四）Redis 和注销

## 前言

上一章我们讲解了生成 JWT ，并实现了用户登录的接口，由于 JWT 的无状态性，只要 JWT 还未过有效期，那么该 JWT 就一直会被服务器认为是有效的，这就会引发一些安全问题。
例如上一章我们设置 JWT 的过期时间为 4 个小时，那么不论用户是关闭浏览器，或者手动退出登录，该 JWT 都是不会失效的，而我们希望当用户退出登录后，当前用户的 JWT 就失效，本章将通过 Redis 来实现这个功能。
关于 Redis：Redis 是什么本文就不再赘述，笔者是在 docker 中安装的 Redis，具体可看我博客的这篇文章：[使用 docker 创建 Redis 服务](https://bald3r.wang/2023/07/27/%E4%BD%BF%E7%94%A8-docker-%E5%88%9B%E5%BB%BA-Redis-%E6%9C%8D%E5%8A%A1/)

## 实现思路

思路参考[这篇文章](https://juejin.cn/post/7160936006517014558)，具体是：

1. 生成 JWT 后（这里生成 JWT 时不设置过期时间），将该 token 存入 Redis 中，并设置 Redis 的过期时间为 4 小时（在 Redis 中设置过期时间，间接的控制了 JWT 的有效时间）
2. 服务器通过删除 Redis 中的 JWT 来实现作废
3. 用户请求接口时，取出 headers 中的 JWT，判断 JWT 自身是否过期，如果没有过期则与 Redis 中的 JWT 进行比较
4. 如果 Redis 中并没有一致的 JWT，则说明该 JWT 被服务器作废，如果找到了一致的 JWT，说明该 JWT 仍然有效，此时重置 Redis 中的过期时间为 4 小时。

这里的意思是，用户登录获取了 JWT 后，只要用户在 4 小时内发请求，则会不断刷新 Redis 中的过期时间，相当于不断给 JWT 续期。当出现以下三种情况：

- 登出
- JWT 被服务器作废
- 超过 4 小时没有发请求

JWT 就会被服务器判定为失效。

## 修改现有代码

现在我们的 JWT 由 Redis 来控制，因此我们在签发 JWT 时，不设置过期时间。

```ts
// src/auth/auth.module.ts
...

const jwtModule = JwtModule.registerAsync({
  inject: [ConfigService],
  useFactory: async (configService: ConfigService) => ({
    secret: configService.get('JWT_SECRET') ?? 'secret',
    signOptions: {
      // expiresIn: configService.get('JWT_EXPIRES_IN') ?? '10m',
    },
  }),
});

...
```

在`.env.local`文件中，配置 Redis 相关的环境变量：

```
...

JWT_SECRET=superNB
JWT_EXPIRES_IN=60000   

REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_DB=1
REDIS_PASSWORD=123456
```

注意这里的`JWT_EXPIRES_IN`变成了数字，是`1 * 60 * 1000`毫秒，即 1 分钟，为什么这么改下文会说明。

## 在 NestJS 中引入 Redis

安装依赖：

```bash
pnpm install @nestjs/cache-manager cache-manager cache-manager-redis-yet redis -S
pnpm install @types/cache-manager -D
```

这里注意，我们安装的是`cache-manager-redis-yet`这个包，网上大多数教程安装的是`cache-manager-redis-store`这个包，后者目前配合 TS 使用有点问题，详见我的另一篇文章：[Nestjs v10 中使用 Redis 作为 CacheStore 的坑](https://bald3r.wang/2023/06/28/Nestjs-v10-%E4%B8%AD%E4%BD%BF%E7%94%A8-Redis-%E4%BD%9C%E4%B8%BA-CacheStore-%E7%9A%84%E5%9D%91/)

创建目录和文件：

```bash
nest g mo db/redis
nest g service db/redis
```

编辑`/src/db/redis/redis.module.ts`文件：

```ts
import { Module, Global } from '@nestjs/common';
import { CacheModule } from '@nestjs/cache-manager';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { redisStore } from 'cache-manager-redis-yet';
import type { RedisClientOptions } from 'redis';
import { RedisService } from './redis.service';

@Global() // 这里我们使用@Global 装饰器让这个模块变成全局的
@Module({
  imports: [
    CacheModule.registerAsync<RedisClientOptions>({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: async (configService: ConfigService) => {
        const store = await redisStore({
          socket: {
            host: configService.get<string>('REDIS_HOST'),
            port: configService.get<number>('REDIS_PORT'),
          },
          ttl: configService.get<number>('REDIS_TTL'),
          database: configService.get<number>('REDIS_DB'),
          password: configService.get<string>('REDIS_PASSWORD'),
        });
        return {
          store,
        };
      },
    }),
  ],
  providers: [RedisService],
  exports: [RedisService],
})
export class RedisModule {}
```

然后我们编辑`redis.service.ts`来实现读写的方法：

```ts
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Inject, Injectable } from '@nestjs/common';
import { Cache } from 'cache-manager';

@Injectable()
export class RedisService {
  constructor(@Inject(CACHE_MANAGER) private readonly cacheManager: Cache) {}

  async get<T>(key: string): Promise<T> {
    return await this.cacheManager.get(key);
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    return await this.cacheManager.set(key, value, ttl);
  }
}
```

在`app.module.ts`中注册`RedisModule`，如果是通过`nest g mo`命令生成的`RedisModule`，那 NestJS 会自动在`app.module.ts`中注册。并且由于我们使用了`@Global()`装饰器，我们在其他模块中使用时，不需要再在`module`中注册。

## 签发 JWT 时存入 Redis

```ts
// src/auth/auth.service.ts

import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { JwtService } from '@nestjs/jwt';
import { RedisService } from 'src/db/redis/redis.service';
import { User } from 'src/user/entities/user.entity';

@Injectable()
export class AuthService {
  constructor(
    private jwtService: JwtService,
    private redisService: RedisService,
    private readonly configService: ConfigService,
  ) {}

  async login(user: Partial<User>) {
    const payload = { username: user.username, id: user.id };

    const access_token = this.jwtService.sign(payload);

    await this.redisService.set(
      `token_${user.id}`,
      access_token,
      this.configService.get('JWT_EXPIRES_IN'), // 注意这里
    );

    return {
      access_token,
      type: 'Bearer',
    };
  }
}

```

可以看到，我们在调用`this.redisService.set()`函数时，传入的第三个参数为`this.configService.get('JWT_EXPIRES_IN')`，从`redis.service.ts`文件中我们不难发现，这里的第三个参数对应的是`ttl`，即 Redis 中这条数据的过期时间，在当前场景下，就是 JWT 的有效时间，因此直接从环境变量中读取`JWT_EXPIRES_IN`的值，由于这里的`ttl`的单位是毫秒，我们在上文的编辑`.env.local`文件时，将`JWT_EXPIRES_IN`值进行了改动，改为了`60000`毫秒，即 60 秒，设置这么短是为了测试方便看出效果，大家可以根据实际情况进行调整。



测试一下是否生效：

```bash
curl --location --request POST 'http://localhost:3000/auth/login' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'username=wang' \
--data-urlencode 'password=123456'
```

响应：

```json
{
    "code": 0,
    "message": "请求成功",
    "data": {
        "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IndhbmciLCJpZCI6MTQsImlhdCI6MTY5MDQyOTM1NH0.bmkZ5PPeZTIRyzlppmvlI3SVcQTx3b0aRHtt5ZOXMiI",
        "type": "Bearer"
    }
}
```

通过 Redis 图形化管理器看一下我们的 Redis 数据库：

![image-20230727114736055](https://img.bald3r.wang/img/image-20230727114736055.png)

可以看到，已经成功将 JWT 存入了 Redis，过期时间也与我们设置的 1 分钟一致。

1 分钟之后，Redis 自动删除了这条数据：

![image-20230727114812352](https://img.bald3r.wang/img/image-20230727114812352.png)



## 请求时进行校验

校验 JWT 的逻辑写在策略中，因此我们对策略进行修改：

```ts
// /src/global/strategy/jwt.strategy.ts

import { PassportStrategy } from '@nestjs/passport';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { ExtractJwt, Strategy } from 'passport-jwt';
import type { StrategyOptions } from 'passport-jwt';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import type { Request } from 'express';
import { User } from 'src/user/entities/user.entity';
import { RedisService } from 'src/db/redis/redis.service';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    @InjectRepository(User) private readonly userRepository: Repository<User>,
    private readonly configService: ConfigService,
    private readonly redisService: RedisSerivce,  // 注意点 ①
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get('JWT_SECRET') ?? 'secret',
      passReqToCallback: true,		// 注意点 ②
    } as StrategyOptions);
  }

  async validate(req: Request, payload: User) { // 注意点 ③
    const token = ExtractJwt.fromAuthHeaderAsBearerToken()(req);
    const existUser = await this.userRepository.findOne({
      where: { id: payload.id },
    });
    const cacheToken = await this.redisService.get(`token_${existUser.id}`);
    if (!cacheToken) throw new UnauthorizedException('token已过期');

    if (token !== cacheToken) throw new UnauthorizedException('token不正确');

    if (!existUser) throw new UnauthorizedException('token验证失败');

    await this.redisService.set(
      `token_${existUser.id}`,
      token,
      this.configService.get('JWT_EXPIRES_IN'),
    );  // 注意点 ④

    return existUser;
  }
}

```

注意点：

1. 在①处引入 RedisService
2. ②处需要将`passReqToCallback`设置为`true`，作用是将请求传递给下面的`validate()`函数，可以看到，③处的`validate()`函数接受的第一个参数就是`req:Request`，否则的话`validate()`函数是拿不到请求的，也就不能从请求头中拿到 JWT
3. ④处是刷新了 Redis 中 JWT 的持续时间，意思就是用户只要发送了请求并且 JWT 验证通过后，就会刷新 JWT 的有效期，达到续期的目的。在本例中，用户一分钟没有操作，JWT 就会过期

等待一分钟后重新请求：

```bash
curl --location --request GET 'http://localhost:3000/user' \
--header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IndhbmciLCJpZCI6MTQsImlhdCI6MTY5MDQzMDM1Mn0._oUGBOO8ycgyjnU1sJLEAmN4Q-B2EsK2wnuL4NHhUks' 
```

```json
{
    "code": 401,
    "message": "token已过期",
    "content": {}
}
```

校验成功。



## 注销接口

接下来我们就通过删除 Redis 中的 key 的方式，实现注销用户登录的接口。

在`redis.service.ts`中新增删除数据的方法：

```ts
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Inject, Injectable } from '@nestjs/common';
import { Cache } from 'cache-manager';

@Injectable()
export class RedisService {
  constructor(@Inject(CACHE_MANAGER) private readonly cacheManager: Cache) {}

  async get<T>(key: string): Promise<T> {
    return await this.cacheManager.get(key);
  }

  async set(key: string, value: any, ttl?: number): Promise<void> {
    return await this.cacheManager.set(key, value, ttl);
  }

  async del(key: string): Promise<void> {
    return await this.cacheManager.del(key);
  }
}

```

在`auth`模块中，新增相应逻辑：

```ts
// /src/auth/auth.controller.ts

import ...

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}
  
	...
  
  @Delete('logout')
  logout(@Req() req: Request) {
    return this.authService.logout(req.user);
  }
}

// /src/auth/auth.service.ts

import ...

@Injectable()
export class AuthService {
  constructor(
  	...
    private redisService: RedisService,
  ) {}

	...

  async logout(user: Partial<User>) {
    await this.redisService.del(`token_${user.id}`);
  }
}


```

用户在`DELETE /auth/logout`接口后，请求其他需要身份认证接口时，都会报`token已过期`错误。



## 后记

本章中的例子有一些限制，比如只支持一个 JWT，也就是说，用户只能在一处进行登录，例如用户网页登录了，然后又用客户端登录，那先登录的网页就会失效，因为 Redis 中只存一条 JWT，并且已经被客户端登录时签发的 JWT 替换了。



[Nest学习系列博客代码仓库 (github.com)](https://github.com/baIder/nest-demo)

[冷面杀手的个人站 (bald3r.wang)](https://bald3r.wang/)

[NestJS 相关文章](https://bald3r.wang/tags/NestJS/)

