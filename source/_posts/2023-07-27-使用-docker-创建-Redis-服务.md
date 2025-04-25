---
title: 使用 docker 创建 Redis 服务
date: 2023-07-27 10:33:59
index_img: https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/20230727103535.png
categories:
  - docker
tags:
  - docker
---
# 使用docker创建Redis服务

## 配置docker镜像源

```json
"registry-mirrors": [
  "https://4346hkfk.mirror.aliyuncs.com",
  "https://docker.mirrors.ustc.edu.cn",
  "https://hub-mirror.c.163.com",
  "https://reg-mirror.qiniu.com"
]
```

## 拉取镜像

```bash
docker pull redis
```

## 创建配置文件

redis并不会创建`redis.conf`配置文件，而在启动容器时，如果宿主机以及容器内都没有配置文件，docker会自动创建，但是docker创建的`redis.conf`是目录，并不是文件，因此我们需要提前创建这个文件。

```bash
mkdir -p ~/redis/conf

touch ~/redis/conf/redis.conf
```

## 配置Redis

编辑`redis.conf`

```bash
vim ~/redis/conf/redis.conf
```

设置以下配置：

```
appendonly yes       # 数据持久化
protected-mode no     # 外部网络访问
bind 0.0.0.0       # 所有ip都可以访问
requirepass 123456    # 设置访问密码
```

## 启动容器

```bash
docker run --name redis -p 6379:6379 -v ~/redis/data:/data -v ~/redis/conf/redis.conf:/etc/redis/redis.conf -d redis:latest redis-server /etc/redis/redis.conf
```

## 进入容器

```bash
docker exec -it redis bash
--------------------------
root@0d0142d45f55:/data# redis-cli -a 123456
Warning: Using a password with '-a' or '-u' option on the command line interface may not be safe.
127.0.0.1:6379> set ping pong
OK
127.0.0.1:6379>
```

通过redis桌面管理工具查看是否生效：

![image-20230707114104329](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image-20230707114104329.png)

说明redis运行成功！
