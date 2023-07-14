---
title: NestJS 入门（三）用户登录与JWT
date: 2023-07-14 15:25:41
index_img: https://img.bald3r.wang/img/20230712153658.png
categories:
  - Node.js
  - NestJS
tags:
  - Node.js
  - NestJS
---
# NestJS 入门（三）用户登录与JWT

## 前言

本文主要探讨在 NestJS 中实现登录功能并签发 JWT Token ，使用的库有：

- node.bcrypt.js
- passport.js
- @nestjs/jwt

## 加密用户密码

目前我们的数据库中的密码是明文存储的，明显是极不安全的，因此我们这里使用第三方库来对密码进行加密，然后再存入数据库中。

首先我们安装库：

```bash
pnpm i -S bcrypt 
pnpm i -D @types/bcrypt
```

前端会将用户的`username`和`password`传给后端，然后后端再将`password`进行加密，最后存入数据库。TypeORM 提供一个装饰器`@BeforeInsert`，它的功能是在数据插入数据库前执行一个函数，符合我们现在的需求。因此接下来我们需要修改`user.entity.ts`：

```ts
// user.entity.ts

import { BeforeInsert, ... , PrimaryGeneratedColumn } from 'typeorm';
import * as bcrypt from 'bcrypt';

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

	...

  @BeforeInsert()
  async hashPassword() {
    if (this.password) this.password = bcrypt.hashSync(this.password, 10);
  }
}

```

此时我们重新创建一个用户：

```bash
curl --location --request POST 'http://localhost:3000/user/' \
--data-urlencode 'username=袁洋' \
--data-urlencode 'password=123456'
```

```json
{
    "code": 0,
    "message": "请求成功",
    "data": {
        "username": "袁洋",
        "password": "$2b$10$Q4Ra7wjNSBCMVKHtbRUf4.rc.jr.wXSvolAI8IAJppUU8LB0AMgvW",
        "id": 13,
        "created_at": "2023-07-13T00:51:13.030Z",
        "updated_at": "2023-07-13T00:51:13.030Z"
    }
}
```

查看数据库：

<img src="https://img.bald3r.wang/img/image-20230713165229514.png" alt="image-20230713165229514" style="zoom:50%;" />

可以看到数据库中的密码字段也已经更新。

细心的读者可能会发现，返回的数据中包含`password`字段，而大多数情况下不需要返回这个字段，因此需要剔除。

剔除有两种方法：

1. 拿到用户数据后，剔除`password`字段，再将其他字段返回。
2. 从数据库中读取用户数据时，就不读取`password`字段。

本文选择第二种方式。

修改`user.entity.ts`：

```ts
...

@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ name: 'account', unique: true })
  username: string;

  @Column({ select: false })     // 增加了 select: false
  password: string;

	...
}

```

该选项会在查表时跳过当前字段。

测试效果：

```bash
curl --location --request GET 'http://localhost:3000/user/1'
```

响应：

```json
{
    "code": 0,
    "message": "请求成功",
    "data": {
        "id": 1,
        "username": "孙明",
        "created_at": "2023-07-12T23:53:01.321Z",
        "updated_at": "2023-07-12T23:53:01.321Z"
    }
}
```

可以看到结果中已经没有`password`字段。

## 登录接口

`passport.js`是 Node.js 中非常著名的一个用于做身份认证的包，它主要依靠策略（Strategy）来进行验证，因此我们还需要一个策略。在本次实践中，我们实现的是本地身份验证，因此我们使用`passport-local`这个策略。

安装依赖：

```bash
pnpm i -S @nestjs/passport passport passport-local
pnpm i -D @types/passport @types/passport-local 
```

创建策略文件，由于 NestJS 并没有提供创建策略文件的命令，因此我们需要手动创建文件：

```ts
// /src/global/strategy/local.strategy.ts

import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import type { IStrategyOptions } from 'passport-local';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { compareSync } from 'bcrypt';
import { BadRequestException } from '@nestjs/common';
import { User } from 'src/user/entities/user.entity';

export class LocalStrategy extends PassportStrategy(Strategy) {		// 此处的 Strategy 要引入 passport-local 中的 
  constructor(
    @InjectRepository(User) private readonly userRepository: Repository<User>,	// 将 user 实体注入进来
  ) {
    super({
      usernameField: 'username',	// 固定写法，指定用户名字段，可以为 phone 或 email 等其他字段，不影响
      passwordField: 'password',	// 固定写法，指定密码字段
    } as IStrategyOptions);
  }

  async validate(username: string, password: string): Promise<any> {		// 必须实现一个 validate 方法
    const user = await this.userRepository
      .createQueryBuilder('user')
      .addSelect('user.password')
      .where('user.username=:username', { username })
      .getOne();

    if (!user) throw new BadRequestException('用户不存在');

    if (!compareSync(password, user.password))
      throw new BadRequestException('密码错误');

    return user;
  }
}

```

这里我们导出了一个类`LocalStrategy`，继承自`PassportStrategy`，这个类首先需要指明两个字段`usernameField`和`passwordField`，一般来说用户登录都会提供至少两个字段，例如用户名（username）和密码（password），或者电子邮箱（email）和密码（password）等等，我们需要告知我们的策略，从请求的`body`中取哪两个字段用于验证。在本例中，我们使用的是`username`和`password`。

策略还必须实现一个方法`validate()`，这个方法会接受我们上面指定的两个字段作为参数，然后就需要查表，查出用户名对应的密码，进行比较。

**注意，由于我们在实体中设置了`password`字段的 `select : false`，因此我们使用`find()`方法是不会返回`password`字段的，因此我们需要使用`createQueryBuilder()`方法创建一个查询命令，再通过`addSelect()`方法手动将`password`字段添加上，这样查询到的数据中就会包含我们所需的`password`字段。**

创建好了策略，我们还需要一个登录接口，一般来说我们的登录地址为`/auth/login`，因此我们创建对应的文件：

```bash
nest g mo auth   
nest g co auth
```

```ts
// auth.module.ts

import { Module } from '@nestjs/common';
import { AuthController } from './auth.controller';
import { LocalStrategy } from 'src/global/strategy/local.strategy';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from 'src/user/entities/user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [AuthController],
  providers: [LocalStrategy],
})
export class AuthModule {}

// auth.controller.ts

import { Controller, Post, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller('auth')
export class AuthController {
  @UseGuards(AuthGuard('local'))
  @Post('login')
  login() {
    return 'login';
  }
}
```

测试一下：

```bash
curl --location --request POST 'http://localhost:3000/auth/login' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'username=wang' \
--data-urlencode 'password=123456'
```

响应成功：

```json
{
    "code": 0,
    "message": "请求成功",
    "data": "login"
}
```

响应失败：

```json
// 用户名输入错误
{
    "code": 400,
    "message": "用户不存在",
    "content": {}
}

// 密码输入错误
{
    "code": 400,
    "message": "密码错误",
    "content": {}
}
```



## 签发 JWT Token

一般来说，登录成功之后会有两种记录登录状态的方式，一种是 Session ，一种是 Token ，本例中使用 JWT Token 。关于 JWT Token ，我也写了[一篇文章](https://bald3r.wang/2023/05/18/登录认证与JWT/)，感兴趣的读者可以移步[我的博客](https://bald3r.wang/)查看。

安装依赖：

```bash
pnpm i -S @nestjs/jwt
```

修改`auth`模块：

```ts
import { Module } from '@nestjs/common';
import { AuthController } from './auth.controller';
import { LocalStrategy } from 'src/global/strategy/local.strategy';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from 'src/user/entities/user.entity';
import { JwtModule } from '@nestjs/jwt';

const jwtModule = JwtModule.register({
  secret: 'suibianshenme',
  signOptions: { expiresIn: '4h' },
});

@Module({
  imports: [TypeOrmModule.forFeature([User]), jwtModule],
  controllers: [AuthController],
  providers: [LocalStrategy],
  exports: [jwtModule],
})
export class AuthModule {}

```

添加`auth.service.ts`，分离登录逻辑：

```bash
nest g service auth
```

修改`auth.controller.ts`：

```ts
import { Controller, Post, Req, UseGuards } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { AuthService } from './auth.service';
import type { Request } from 'express';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}
  @UseGuards(AuthGuard('local'))
  @Post('login')
  login(@Req() req: Request) {
    return this.authService.login(req.user);
  }
}
```

这里的`req.user`是我们的策略`local.strategy.ts`，最后验证成功后`return user`挂载上去的。

修改`auth.service.ts`：

```ts
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { User } from 'src/user/entities/user.entity';

@Injectable()
export class AuthService {
  constructor(private jwtService: JwtService) {}

  async login(user: Partial<User>) {
    const payload = { username: user.username, id: user.id };

    const access_token = this.jwtService.sign(payload);

    return {
      access_token,
      type: 'Bearer',
    };
  }
}
```

测试一下：

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
        "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IndhbmciLCJpZCI6MTQsImlhdCI6MTY4OTMwNjc0NywiZXhwIjoxNjg5MzIxMTQ3fQ.QrV8vjQatf7KYaM6fwckNSuNC2A08IUFyGkJzMehzaw",
        "type": "Bearer"
    }
}
```

至此，实现签发 JWT token 。



## 验证 JWT Token

用户在请求需要身份验证的接口时，会在请求的`headers`中增加一个字段`Authorization : Bearer {token}`，接下来我们就从请求头中取出 token 并进行验证。

我们使用的`passport.js`也提供了相应的策略`passport-jwt`，帮助我们进行验证。

安装依赖：

```bash
pnpm i -S passport-jwt
pnpm i -D @types/passport-jwt
```

创建新的策略：

```ts
// /src/global/strategy/jwt.strategy.ts

import { PassportStrategy } from '@nestjs/passport';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { ExtractJwt, Strategy } from 'passport-jwt';
import type { StrategyOptions } from 'passport-jwt';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { User } from 'src/user/entities/user.entity';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {		// 这里的 Strategy 必须是 passport-jwt 包中的
  constructor(
    @InjectRepository(User) private readonly userRepository: Repository<User>,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: 'suibianshenme',
    } as StrategyOptions);
  }

  async validate(payload: User) {
    const existUser = await this.userRepository.findOne({
      where: { id: payload.id },
    });

    if (!existUser) throw new UnauthorizedException('token验证失败');

    return existUser;
  }
}
```

策略的内容与`local`策略基本一致，通过包提供的`ExtractJwt.fromAuthHeaderAsBearerToken()`方法可以自动从`headers`中提取`Authorization`中的 token ，并且会自动去除开头的`Bearer `前缀。注意这里的`secretOrKey`需要和签发时的`secret`一致。

策略必须实现一个方法`validate()`，其中的参数`payload`是我们签发的 JWT Token 中的`payload`部分：

<img src="https://img.bald3r.wang/img/image-20230714135538574.png" alt="image-20230714135538574" style="zoom:50%;" />

所以`payload`这里其实是一个对象，包含了`username`和`id`字段。

创建好策略后，我们还需要注册这个策略。

例如我们给获取用户信息接口`GET /user/{id}`加入 Token 验证：

```ts
// user.module.ts

import { Module } from '@nestjs/common';
import { UserService } from './user.service';
import { UserController } from './user.controller';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './entities/user.entity';
import { JwtStrategy } from 'src/global/strategy/jwt.strategy';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UserController],
  providers: [UserService, JwtStrategy],	// 将策略加入 providers 数组
})
export class UserModule {}

// user.controller.ts

import {
  ...,
  UseGuards,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller('user')
export class UserController {
  constructor(private readonly userService: UserService) {}
  
  ...

  @UseGuards(AuthGuard('jwt'))
  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.userService.findOne(+id);
  }

	...
}

```

测试一下：

```bash
curl --location --request GET 'http://localhost:3000/user/1' 
```

请求失败：

```json
{
    "code": 401,
    "message": "Unauthorized",
    "content": {}
}
```

我们先登录，然后将得到的 JWT Token 加入到`headers`中，重新请求：

```bash
curl --location --request GET 'http://localhost:3000/user/1' \
--header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IndhbmciLCJpZCI6MTQsImlhdCI6MTY4OTMxNDY3NywiZXhwIjoxNjg5MzI5MDc3fQ.KMXnv3X_CIZHwRdnFxMPIbs_H5_mMKpE3oDqcMICWh8' 
```

请求成功：

```json
{
    "code": 0,
    "message": "请求成功",
    "data": {
        "id": 1,
        "username": "孙明",
        "created_at": "2023-07-12T23:53:01.321Z",
        "updated_at": "2023-07-12T23:53:01.321Z"
    }
}
```

但是如果对每个接口都加一个`@UseGuard(AuthGuard('jwt'))`显然是繁琐且重复的，绝大多数接口都是需要验证身份的，只有诸如登录一类的接口是不需要认证的，因此我们下一步就是全局注册。

## 将 Token 验证应用到全局

首先我们需要理清思路：

- 现有的`AuthGuard('jwt')`无法满足需求，我们需要定制
- 有个别接口不需要验证，需要排除/标记 

### 做排除

我们可以维护一个白名单，在策略中验证请求的 url 是否在白名单中，如果是则跳过验证。这里笔者就不展开了。

### 做标记

我们自定义一个装饰器`@Public`来标记接口是否为公共接口，所有被标记的接口都可以不需要身份验证。

在`/src/global/decorator`目录下创建一个`public.decorator.ts`：

```ts
import { SetMetadata } from '@nestjs/common';

export const IS_PUBLIC_KEY = 'isPublic';
export const Public = () => SetMetadata(IS_PUBLIC_KEY, true);
```

这里主要是使用了`SetMetadata()`方法，给接口设置了一个元数据（Metadata）`isPublic : true`

然后给接口加上这个标记：

```ts
// auth.controller.ts

...

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}
  @Public()
  @UseGuards(AuthGuard('local'))
  @Post('login')
  login(@Req() req: Request) {
    return this.authService.login(req.user);
  }
}
```

删除我们之前加在`user.controller.ts`中的代码：

```ts
// user.controller.ts

import {
  ...,
  UseGuards,
} from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller('user')
export class UserController {
  constructor(private readonly userService: UserService) {}
  
  ...

//@UseGuards(AuthGuard('jwt'))
  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.userService.findOne(+id);
  }

	...

```

### 定制一个 Guard

> 在 Nest.js 中，Guard（守卫）是一种用于保护路由和执行权限验证的特殊类型组件。它允许您在请求到达路由处理程序之前对请求进行拦截，并根据特定条件来允许或拒绝请求的访问。
>
> Guard 可以用于实现各种身份验证和授权策略，例如基于角色的访问控制、JWT 验证、OAuth 认证等。它们可以在路由级别或处理程序级别应用，以确保请求的安全性和合法性。
>
> Guard 类必须实现 `CanActivate` 接口，并实现 `canActivate()` 方法来定义守卫的逻辑。在该方法中，您可以根据请求的特征、用户信息、权限等进行验证，并返回一个布尔值来表示是否允许请求继续执行。

在`/src/global/guard`目录下创建一个`jwt-auth.guard.ts`：

```ts
import type { ExecutionContext } from '@nestjs/common';
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { Reflector } from '@nestjs/core';
import type { Observable } from 'rxjs';
import { IS_PUBLIC_KEY } from '../decorator/public.decorator';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const isPublic = this.reflector.getAllAndOverride<boolean>(IS_PUBLIC_KEY, [
      context.getHandler(),
      context.getClass(),
    ]);

    if (isPublic) return true;

    return super.canActivate(context);
  }
}
```

这里的 Guard 必须实现一个`canActive()`方法，本例中，我们通过`Reflector`拿到了通过装饰器设置的元数据`isPublic`，如果其为`true`，继续执行请求的逻辑，如果为`false`，将请求传递给其他代码执行。

在`app.module.ts`中注册这个 Guard：

```ts
import { Module } from '@nestjs/common';
import { AppService } from './app.service';
...
import { JwtAuthGuard } from './global/guard/jwt-auth.guard';
import { APP_GUARD } from '@nestjs/core';

@Module({
  ...
  providers: [
    AppService,
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
    },
  ],
})
export class AppModule {}
```

这时我们重新请求`GET /user/{id}`和`GET /user`，都会提示未验证，但是我们请求`POST /auth/login`是没问题的，至此 JWT 验证部分就结束了。



## 环境变量

截至目前，我们的项目中有两个敏感信息是明文写在代码中的，一个是我们连接数据库的信息，一个是我们签发 JWT Token 的密钥。出于安全性考虑，我们一般会将这些数据写在环境变量中，让我们的代码运行时从环境变量中读取。

创建`.env.local`文件，用于本地开发，创建`.env.prod`用于生产环境，这里以`.env.local`为例：

```ts
// .env.local

DB_HOST=localhost
DB_PORT=3306
DB_USERNAME=root
DB_PASSWORD=123456
DB_DATABASE=nest-demo

JWT_SECRET=superNB
JWT_EXPIRES_IN=10m
```

在根目录`/`下新建`config`目录，用来存放我们读取环境变量的代码，并在该目录下创建文件`envConfig.ts`：

```ts
// envConfig.ts

import * as fs from 'node:fs'
import * as path from 'node:path'

const isProd = process.env.NODE_ENV === 'production'

function parseEnv() {
  const localEnv = path.resolve('.env.local')
  const prodEnv = path.resolve('.env.prod')

  if (!fs.existsSync(localEnv) && !fs.existsSync(prodEnv))
    throw new Error('缺少环境配置文件')

  const filePath = isProd && fs.existsSync(prodEnv) ? prodEnv : localEnv
  return { path: filePath }
}

export default parseEnv()
```

安装依赖：

```bash
pnpm i -S @nestjs/config
```

然后在`app.module.ts`中全局注册我们的`config`：

```ts
// app.module.ts

import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import envConfig from 'config/envConfig';
...

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: [envConfig.path],
    }),
  ...
  ],
  ...
})
export class AppModule {}
```

然后也是在`app.module.ts`中将我们数据库信息替换成环境变量中读取的信息：

```ts
// app.module.ts

import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';
...

@Module({
  imports: [
    ...
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: async (configService: ConfigService) => ({
        type: 'mysql',
        host: configService.get<string>('DB_HOST') ?? 'localhost',
        port: configService.get<number>('DB_PORT') ?? 3306,
        username: configService.get<string>('DB_USERNAME') ?? 'root',
        password: configService.get<string>('DB_PASSWORD') ?? '123456',
        database: configService.get<string>('DB_DATABASE') ?? 'nest-demo',
        synchronize: true,
        retryDelay: 500,
        retryAttempts: 10,
        autoLoadEntities: true,
      }),
    }),
  ],
  ...
})
export class AppModule {}
```

将原本代码中签发和验证 JWT 处的密钥进行替换：

```ts
// /src/auth/auth.module.ts

import { JwtModule } from '@nestjs/jwt';
import { ConfigService } from '@nestjs/config';
...

const jwtModule = JwtModule.registerAsync({
  inject: [ConfigService],
  useFactory: async (configService: ConfigService) => ({
    secret: configService.get('JWT_SECRET') ?? 'secret',
    signOptions: {
      expiresIn: configService.get('JWT_EXPIRES_IN') ?? '10m',
    },
  }),
});
...
export class AuthModule {}

```

```ts
// /src/global/strategy/jwt.strategy.ts

import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import type { StrategyOptions } from 'passport-jwt';
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
...

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(
    ...
    private readonly configService: ConfigService,
  ) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get('JWT_SECRET') ?? 'secret',
    } as StrategyOptions);
  }

  ...
}

```



## 后记

笔者也是刚刚接触 Node ，目前还存在诸多不足，如果文章中有任何错误，欢迎在评论区批评指正。



[冷面杀手的个人站 (bald3r.wang)](https://bald3r.wang/)

[NestJS 相关文章](