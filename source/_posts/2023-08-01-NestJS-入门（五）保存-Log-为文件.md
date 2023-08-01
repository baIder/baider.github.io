---
title: NestJS 入门（五）保存 Log 为文件
date: 2023-08-01 14:39:52
index_img: https://img.bald3r.wang/img/20230712153658.png
categories:
  - Node.js
  - NestJS
tags:
  - Node.js
  - NestJS
---
# NestJS 入门（五）保存 Log 为文件



## 前言

到这里，我们的后台项目的骨架就基本完成了，在最后我们需要给项目增加一点点“记忆”——将项目中的部分 log 信息保存到本地文件中。本章主要使用`winston`来将日志记录到文件中。

一般来说，为了方便排查问题，log 中会记录请求的信息，报错的信息，和服务器的返回信息，分别对应了 NestJS 中的中间件、过滤器和拦截器，因此我们主要改造这三个部分。



## 安装 `winston`

`winston`是 Node.js 中非常流行的日志记录库，可以通过配置将日志记录到控制台、文件、数据库等不同目标中。

安装：

```bash
pnpm install --save nest-winston winston winston-daily-rotate-file
```

我们安装了三个包，一个是主角`winston`，一个是`nest-winston`，这个包将`winston`封装成了 NestJS 的 Module，就不需要我们二次封装了，还有一个是`winston-daily-rotate-file`，这个包主要是用来做日志文件的归档的，可以自动将日志按时间或日期等规则进行分割，避免日志都记录在一个巨大的文件中。



## 在项目中引入 `winston`

类似与 TypeORM 或 Redis，我们也需要在`app.module.ts`中注册`winston`。

```ts
// app.module.ts

...
import { WinstonModule } from 'nest-winston';
import 'winston-daily-rotate-file';
import { transports } from 'winston';

@Module({
  imports: [
    ...
    WinstonModule.forRoot({
      transports: [
        new transports.DailyRotateFile({
          dirname: `logs`,
          filename: `%DATE%.log`,
          datePattern: 'YYYY-MM-DD',
          zippedArchive: true,
          maxSize: '20m',
          maxFiles: '14d',
        }),
      ],
    }),
  ],
  ...
})
export class AppModule {}

```

这里我们将 log 文件命名为`%DATE%.log`，并存储在根目录的`logs`中。`%DATE%`最终会被`datePattern`的值替换，也就是说，最后 log 文件会以`logs/2023-08-01.log`的形式保存，可以根据自己的需要自定义。

`zippedArchive`表示是否需要用 gzip 方式压缩文件，默认为`false`；

`maxSize`和`maxFiles`很好理解，设置单个日志文件最大为 20MB，日志文件最长保存 14 天。

此时保存并重启项目，可以看到`logs`目录已经生成了。

```
.
├── README.md
├── config
├── dist
├── logs							//	在这里
├── nest-cli.json
├── node_modules
├── package.json
├── pnpm-lock.yaml
├── src
├── test
├── tsconfig.build.json
└── tsconfig.json

7 directories, 6 files
```



## 添加中间件

我们需要记录每个请求的信息，比如请求的 IP，请求的 URL 等，我们可以通过中间件来实现这个功能。

新建一个中间件：

```bash
nest g mi logger global/middleware
```

编写我们的逻辑：

```ts
// /src/global/middleware/logger.middleware.ts

import { Inject, Injectable, NestMiddleware } from '@nestjs/common';
import { NextFunction, Request, Response } from 'express';
import { WINSTON_MODULE_PROVIDER } from 'nest-winston';
import { Logger } from 'winston';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  constructor(
    @Inject(WINSTON_MODULE_PROVIDER) private readonly logger: Logger,       // ①
  ) {}
  use(req: Request, res: Response, next: NextFunction) {
    const { method, originalUrl: url, body, query, params, ip } = req;
    this.logger.info('route', {
      req: {
        method,
        url,
        body,
        query,
        params,
        ip,
      },
    });
    next();
  }
}

```

这里我们从请求中取出了`method, originalUrl, body, query, params, ip`并记录在日志中，各位读者也可以根据自己的需要进行修改。

**注意**：在①处，`@Inject()`中别忘记传入`WINSTON_MODULE_PROVIDER`，同时后面的`Logger`也要引入`winston`包中的，因为 NestJS 自身也会导出一个`Logger`，注意区分。

创建了中间件之后，我们需要在`app.module.ts`中应用一下：

```ts
// app.module.ts

import { MiddlewareConsumer, Module, NestModule } from '@nestjs/common';
import { LoggerMiddleware } from './global/middleware/logger/logger.middleware';
...
...
...
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*');
  }
}
```

这里我们将中间件`LoggerMiddleware`应用在`*`即所有路由上。

我们随便请求一个接口，然后打开`logs`目录下 log 文件查看：

```json
// logs/2023-08-01.log

{"level":"info","message":"route","req":{"body":{"password":"123456","username":"wang"},"ip":"::1","method":"POST","params":{"0":"auth/login"},"query":{},"url":"/auth/login"}}
```

没有问题。



## 改造错误过滤器

废话少说，直接开改：

```ts
// /src/global/filter/http-exception.filter.ts

import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  Inject,
} from '@nestjs/common';
import { Response } from 'express';
import { WINSTON_MODULE_PROVIDER } from 'nest-winston';
import { Logger } from 'winston';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  constructor(
    @Inject(WINSTON_MODULE_PROVIDER) private readonly logger: Logger,
  ) {}
  catch(exception: HttpException, host: ArgumentsHost) {
    ...

    const {
      method,
      originalUrl: url,
      body,
      query,
      params,
      ip,
    } = ctx.getRequest();
    this.logger.error(message, {
      status,
      req: { method, originalUrl: url, body, query, params, ip },
    });
  }
}

```

从 22 行到 29 行的代码简单且重复，因此抽离出来封装成一个函数：

```ts
// /src/global/helper/getInfoFromReq.ts

import { Request } from 'express';

export const getInfoFromReq = (req: Request) => {
  const { method, originalUrl: url, body, query, params, ip } = req;
  return {
    method,
    url,
    body,
    query,
    params,
    ip,
  };
};

```

然后重新修改错误过滤器：

```ts
// /src/global/filter/http-exception.filter.ts

import {
  ArgumentsHost,
  Catch,
  ExceptionFilter,
  HttpException,
  Inject,
} from '@nestjs/common';
import { Response } from 'express';
import { WINSTON_MODULE_PROVIDER } from 'nest-winston';
import { Logger } from 'winston';
import { getInfoFromReq } from 'src/global/helper/getInfoFromReq';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  constructor(
    @Inject(WINSTON_MODULE_PROVIDER) private readonly logger: Logger,
  ) {}
  catch(exception: HttpException, host: ArgumentsHost) {
    ...

    this.logger.error(message, {
      status,
      req: getInfoFromReq(ctx.getRequest()),
    });
  }
}

```

**不要忘记中间件中的代码也可以用工具函数`getInfoFromReq()`进行精简：**

```ts
// /src/global/middleware/logger.middleware.ts

import { Inject, Injectable, NestMiddleware } from '@nestjs/common';
import { NextFunction, Request, Response } from 'express';
import { WINSTON_MODULE_PROVIDER } from 'nest-winston';
import { getInfoFromReq } from 'src/global/helper/getInfoFromReq';
import { Logger } from 'winston';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  constructor(
    @Inject(WINSTON_MODULE_PROVIDER) private readonly logger: Logger,
  ) {}
  use(req: Request, res: Response, next: NextFunction) {
    this.logger.info('route', {
      req: getInfoFromReq(req),
    });
    next();
  }
}
```

这时发现`main.ts`中引用`HttpExceptionFilter`的地方报错了：

![image-20230801141606690](https://img.bald3r.wang/img/image-20230801141606690.png)

我们删除该行代码，改为在`app.module.ts`中注册：

```ts
// main.ts

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';
import { TransformInterceptor } from './global/interceptor/transform/transform.interceptor';
// import { HttpExceptionFilter } from './global/filter/http-exception/http-exception.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  app.useGlobalInterceptors(new TransformInterceptor());
  // app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(3000);
}
bootstrap();
```

```ts
// app.module.ts

...
import { APP_FILTER, APP_GUARD } from '@nestjs/core';
import { HttpExceptionFilter } from './global/filter/http-exception/http-exception.filter';

@Module({
  ...
  providers: [
    AppService,
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
    },
    {
      provide: APP_FILTER,							// 在这里注册
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*');
  }
}

```

测试一下：

```bash
curl --location --request GET 'http://localhost:3000/user/1' 
```

响应：

```json
{
    "code": 401,
    "message": "token已过期",
    "content": {}
}
```

查看我们的 log：

```json
{"level":"info","message":"route","req":{"body":{"password":"123456","username":"wang"},"ip":"::1","method":"POST","params":{"0":"auth/login"},"query":{},"url":"/auth/login"}}
{"level":"info","message":"route","req":{"body":{},"ip":"::1","method":"GET","params":{"0":"user/1"},"query":{},"url":"/user/1"}}
{"level":"error","message":"token已过期","req":{"body":{},"ip":"::1","method":"GET","params":{"id":"1"},"query":{},"url":"/user/1"},"status":401}
```

可以看到，错误也正确的记录了下来。



## 改造响应拦截器

废话少说，开改：

```ts
// /src/global/interceptor/transform.interceptor.ts

import {
  CallHandler,
  ExecutionContext,
  Inject,
  Injectable,
  NestInterceptor,
} from '@nestjs/common';
import { WINSTON_MODULE_PROVIDER } from 'nest-winston';
import { Observable, map } from 'rxjs';
import { Logger } from 'winston';
import { getInfoFromReq } from 'src/global/helper/getInfoFromReq';

@Injectable()
export class TransformInterceptor implements NestInterceptor {
  constructor(
    @Inject(WINSTON_MODULE_PROVIDER) private readonly logger: Logger,
  ) {}
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => {
        this.logger.info('response', {
          responseData: data,
          req: getInfoFromReq(context.switchToHttp().getRequest()),
        });
        return {
          code: 0,
          message: '请求成功',
          data,
        };
      }),
    );
  }
}

```

同样的，我们也要将响应拦截器的注册位置从`main.ts`转移到`app.module.ts`中：

```ts
// main.ts

import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { ValidationPipe } from '@nestjs/common';
// import { TransformInterceptor } from './global/interceptor/transform/transform.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalPipes(new ValidationPipe());
  // app.useGlobalInterceptors(new TransformInterceptor());
  await app.listen(3000);
}
bootstrap();
```

```ts
// app.module.ts

...
import { APP_FILTER, APP_GUARD, APP_INTERCEPTOR } from '@nestjs/core';
import { TransformInterceptor } from './global/interceptor/transform/transform.interceptor';

@Module({
  ...
  providers: [
    AppService,
    {
      provide: APP_GUARD,
      useClass: JwtAuthGuard,
    },
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
    {
      provide: APP_INTERCEPTOR,						// 在这里注册
      useClass: TransformInterceptor,
    },
  ],
})
export class AppModule implements NestModule {
  configure(consumer: MiddlewareConsumer) {
    consumer.apply(LoggerMiddleware).forRoutes('*');
  }
}

```

测试一下：

重新请求了登录接口，查看 log：

```json
{"level":"info","message":"route","req":{"body":{"password":"123456","username":"wang"},"ip":"::1","method":"POST","params":{"0":"auth/login"},"query":{},"url":"/auth/login"}}
{"level":"info","message":"route","req":{"body":{},"ip":"::1","method":"GET","params":{"0":"user/1"},"query":{},"url":"/user/1"}}
{"level":"error","message":"token已过期","req":{"body":{},"ip":"::1","method":"GET","params":{"id":"1"},"query":{},"url":"/user/1"},"status":401}
{"level":"info","message":"route","req":{"body":{"password":"123456","username":"wang"},"ip":"::1","method":"POST","params":{"0":"auth/login"},"query":{},"url":"/auth/login"}}
{"level":"info","message":"response","req":{"body":{"password":"123456","username":"wang"},"ip":"::1","method":"POST","params":{},"query":{},"url":"/auth/login"},"responseData":{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IndhbmciLCJpZCI6MTQsImlhdCI6MTY5MDg3MTE2Nn0.MT5H1Izgh4L9PleNhzfQsVYlVmPcQkxTCpKDnnO2i-s","type":"Bearer"}}
```

可以看到这次请求的信息和响应的信息都符合预期的记录在了文件中。



## 美化 log

现在我们已经可以将一些重要日志记录在本地文件中，但是查看的时候很费劲，一坨一坨的数据辣眼睛，因此笔者决定对日志进行格式化，方便查看。

```ts
// app.module.ts

...
import { format, transports } from 'winston';

@Module({
  imports: [
    ...
    WinstonModule.forRoot({
      transports: [
        new transports.DailyRotateFile({
          dirname: `logs`,
          filename: `%DATE%.log`,
          datePattern: 'YYYY-MM-DD',
          zippedArchive: true,
          maxSize: '20m',
          maxFiles: '14d',
          format: format.combine(
            format.timestamp({
              format: 'YYYY-MM-DD HH:mm:ss',
            }),
            format.printf(
              (info) =>
                `${info.timestamp} [${info.level}] : ${info.message} ${
                  Object.keys(info).length ? JSON.stringify(info, null, 2) : ''
                }`,
            ),
          ),
        }),
      ],
    }),
  ],
  ...
})

...
```

`winston`在配置时可以接受一个`format`参数，自定义格式，通过`winston`包中自带的`format`可以进行很多操作，这里笔者就不展开了，大家可以根据自己的喜好，查阅[文档](https://github.com/winstonjs/winston#formats)，进行自定义。

查看一下效果：

```json
{"level":"info","message":"route","req":{"body":{"password":"123456","username":"wang"},"ip":"::1","method":"POST","params":{"0":"auth/login"},"query":{},"url":"/auth/login"}}
{"level":"info","message":"route","req":{"body":{},"ip":"::1","method":"GET","params":{"0":"user/1"},"query":{},"url":"/user/1"}}
{"level":"error","message":"token已过期","req":{"body":{},"ip":"::1","method":"GET","params":{"id":"1"},"query":{},"url":"/user/1"},"status":401}
{"level":"info","message":"route","req":{"body":{"password":"123456","username":"wang"},"ip":"::1","method":"POST","params":{"0":"auth/login"},"query":{},"url":"/auth/login"}}
{"level":"info","message":"response","req":{"body":{"password":"123456","username":"wang"},"ip":"::1","method":"POST","params":{},"query":{},"url":"/auth/login"},"responseData":{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IndhbmciLCJpZCI6MTQsImlhdCI6MTY5MDg3MTE2Nn0.MT5H1Izgh4L9PleNhzfQsVYlVmPcQkxTCpKDnnO2i-s","type":"Bearer"}}
2023-08-01 14:34:43 [info] : route {
  "req": {
    "method": "POST",
    "url": "/auth/login",
    "body": {
      "username": "wang",
      "password": "123456"
    },
    "query": {},
    "params": {
      "0": "auth/login"
    },
    "ip": "::1"
  },
  "level": "info",
  "message": "route",
  "timestamp": "2023-08-01 14:34:43"
}
2023-08-01 14:34:43 [info] : response {
  "responseData": {
    "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VybmFtZSI6IndhbmciLCJpZCI6MTQsImlhdCI6MTY5MDg3MTY4M30.n-17Fz3On6n1UfOKfRMFzBbXfzbfNQt_oiC-1WXzhB4",
    "type": "Bearer"
  },
  "req": {
    "method": "POST",
    "url": "/auth/login",
    "body": {
      "username": "wang",
      "password": "123456"
    },
    "query": {},
    "params": {},
    "ip": "::1"
  },
  "level": "info",
  "message": "response",
  "timestamp": "2023-08-01 14:34:43"
}

```

这样美化有利有弊，大家可以根据实际情况酌情选择。



## 后记

本系列文章到这里算是完结了，不论是 NestJS 还是本系列文章中提到的各种工具，都有着非常多的功能，笔者也只是一个初学者，也只是浅尝辄止的进行了学习，并分享给大家，后面笔者也会继续分享其他内容。如果有可以改进的地方，欢迎和我交流，如果有错误，还请大家斧正。



[Nest学习系列博客代码仓库 (github.com)](https://github.com/baIder/nest-demo)

[冷面杀手的个人站 (bald3r.wang)](https://bald3r.wang/)

[NestJS 相关文章](https://bald3r.wang/tags/NestJS/)

