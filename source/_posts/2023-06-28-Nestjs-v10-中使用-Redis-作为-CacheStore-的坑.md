---
title: Nestjs v10 中使用 Redis 作为 CacheStore 的坑
date: 2023-06-28 15:42:16
index_img: https://img.bald3r.wang/img/image-20230628145215603.png
categories:
  - Node.js
  - NestJS
tags:
  - Node.js
  - NestJS
  - Redis
  - 踩坑
---
## 开发环境

- Nest.js v10.0.1

    其他 npm 包版本如下

```JSON
"@nestjs/cache-manager": "^2.0.1",
"cache-manager": "^5.2.3",
"cache-manager-redis-store": "^3.0.1",
"cache-manager-redis-yet": "^4.1.2",
"redis": "^4.6.7",
```

## 背景

最近笔者在学习 `node.js`，选择了`Nestjs`作为后端框架，在掘金上找了篇[文章](https://juejin.cn/post/7160936006517014558)（后文简称教程）跟着一起敲，那位作者使用的是`Nestjs v8`的版本，因此在`redis`相关的章节遇到了问题，最后参考了官方文档[官方文档](https://docs.nestjs.com/techniques/caching#different-stores)以及相关npm包的[Github Issue](https://github.com/dabroek/node-cache-manager-redis-store/issues/40)中的内容，暂时解决了。笔者百度了很久`Nestjs`相关的`redis`教程，普遍都比较老了，也基本是大同小异，因此笔者打算写这篇文章作为记录。

## 问题复现

教程中的代码：

```TypeScript
import { ConfigModule, ConfigService } from '@nestjs/config';
import { RedisCacheService } from './redis-cache.service';
import { CacheModule, Module, Global } from '@nestjs/common';
import * as redisStore from 'cache-manager-redis-store';

@Module({
imports: [
CacheModule.registerAsync({
 isGlobal: true,
 imports: [ConfigModule],
 inject: [ConfigService],
 useFactory: async (configService: ConfigService) => {
   return {
     store: redisStore,
     host: configService.get('REDIS_HOST'),
     port: configService.get('REDIS_PORT'),
     db: 0, //目标库,
     auth_pass:  configService.get('REDIS_PASSPORT') // 密码,没有可以不写
   };
 },
}),
],
providers: [RedisCacheService],
exports: [RedisCacheService],
})
export class RedisCacheModule {}
```



发现`useFactory`函数部分报错，是一个TS错误：

![image-20230628145215603](https://img.bald3r.wang/img/image-20230628145215603.png)

## 解决问题？

问题出现的原因是此处`useFactory`函数需要返回一个类型为`Promise<CacheModuleOptions<StoreConfig>> | CacheModuleOptions<StoreConfig>`的值，很显然笔者此时返回的对象有问题。

```TypeScript
export interface CacheModuleAsyncOptions<StoreConfig extends Record<any, any> = Record<string, any>> extends ConfigurableModuleAsyncOptions<CacheModuleOptions<StoreConfig>, keyof CacheOptionsFactory> {
    useExisting?: Type<CacheOptionsFactory<StoreConfig>>;
    useClass?: Type<CacheOptionsFactory<StoreConfig>>;
    useFactory?: (...args: any[]) => Promise<CacheModuleOptions<StoreConfig>> | CacheModuleOptions<StoreConfig>;
    inject?: any[];
    extraProviders?: Provider[];
    isGlobal?: boolean;
}
```



从报错中可以看出，问题基本出在这个`store`上，而这个store的类型是由`cache-manager-redis-store` 这个npm包决定的，因此查看这个包有什么变动。



由于版本的升级，该包已经可以按需引入了，因此我们使用

```TypeScript
import { redisStore } from 'cache-manager-redis-store';
```



同时，写法也有所变化：

```TypeScript
@Module({
  imports: [
    CacheModule.registerAsync({
      ...
      useFactory: async (configService: ConfigService) => {
        return {
          store: redisStore({
            socket: {
              host: configService.get('REDIS_HOST'),
              port: configService.get('REDIS_PORT'),
            },
            database: configService.get('REDIS_DB'),
            password: configService.get('REDIS_PASSWORD'),
          }),
        };
      },
  ...
})
```



发现还是报错，但是错误信息不同：

![image-20230628150303497](https://img.bald3r.wang/img/image-20230628150303497.png)



核心是这句：`Type 'Promise<RedisStore>' is not assignable to type 'string | CacheStoreFactory | CacheStore'.` 

也就是说类型还是不匹配。

查阅[官方文档](https://docs.nestjs.com/techniques/caching#different-stores)，发现文档相当之老，但是有一个Warning引起了笔者的注意：

![image-20230628150800795](https://img.bald3r.wang/img/image-20230628150800795.png)

当场点开了这个issue，让我康康。

## 解决问题！

![image-20230628151022075](https://img.bald3r.wang/img/image-20230628151022075.png)



这个npm包的作者提出了解决方案：`//@ts-ignore`  😅



加入代码后，能正常的跑起来了，但是在写入值的时候，又报错了：

> [Nest] 75156  - 06/28/2023, 3:10:41 PM   ERROR [ExceptionsHandler] store.set is not a function



于是继续看这个issue，发现了解决方案，似乎是和异步有关系

```TypeScript
@Module({
  imports: [
    CacheModule.registerAsync({
    ...
      //@ts-ignore
      useFactory: async (configService: ConfigService) => {
        const store = await redisStore({
          ...
        });
        return {
          store,
        };
      },
    }),
    ...
})
```



这样报错就解决了，redis也能正常使用了



但是笔者作为一个高贵的TS使用者，怎么能接受自己的代码中有`@ts-ignore`出现呢？？？这也太不专业了，正巧，issue中也有老哥跟笔者一个想法，因此「 优化版 」出现了。

```TypeScript
@Module({
  imports: [
    CacheModule.registerAsync({
      ...
      useFactory: async (configService: ConfigService) => {
        const store = await redisStore({
          ...
        });
        return {
          store: store as unknown as CacheStore,
        };
      },
    }),
    ...
})
```



看起来舒服了一点，但是issue中有硬核老哥觉得这还不行，作者一直不改，那就自己改，于是魔改了这个包，发布了另一个包`cache-manager-redis-yet`，这个替换这个包就可以了，两者就最后一个单词不同。

```Bash
cache-manager-redis-store
cache-manager-redis-yet
```



至此问题基本解决了：

```TypeScript
import { redisStore } from 'cache-manager-redis-yet';
import type { RedisClientOptions } from 'redis';
...

@Module({
  imports: [
    CacheModule.registerAsync<RedisClientOptions>({
      ...
      useFactory: async (configService: ConfigService) => {
        const store = await redisStore({
          ...
        });
        return {
          store: store,
        };
      },
    }),
    ...
})
```



## 总结

解决方式有两种（前提是你已经使用了新的写法）：

- 使用`@ts-ignore`直接忽略报错
- 使用`cache-manager-redis-yet`替代原来的`cache-manager-redis-store`

希望笔者的文章能帮到你。

