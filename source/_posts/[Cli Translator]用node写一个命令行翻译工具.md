---
title: 用node写一个命令行翻译工具
date: 2022-08-08 15:29:14
categories:
  - Node.js
tags:
  - Node.js
---

# \[Cli Translator]用node写一个命令行翻译工具

## 前言

这是一个用node和TypeScript写的命令行工具，可以翻译中文或者英文单词，主要是实践用node发请求，目前已经发布[npm](https://www.npmjs.com/package/bald3r-node-cli-translator "npm")和[GitHub](https://github.com/baIder/node-translator "GitHub")

## 使用方法

安装`bald3r-node-cli-translator`

```bash
npm i -g bald3r-node-cli-translator

or

yarn global add bald3r-node-cli-translator
```

然后就可以愉快的在命令行里用`fy 词语`的方式进行翻译了

![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_9pBdmyYb5u.png)

或者英译中

![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_t8ScM-XsgT.png)

## 实现过程

其实主要就是构造一个请求，然后调用某翻译api就行

在构造查询参数时，以前常用的`querystring.stringify()`已经噶了，在node14时弃用了，

![](https://img.bald3r.wang/img/20220807220726.png)

![](https://img.bald3r.wang/img/20220807220523.png)

网上有很多教程教你怎么关闭编辑器的弃用提示，我个人还是比较喜欢尝新的，因此选用了node推荐的`URLSearchParams`

然后解析返回的`response`，得到最终的结果，如果有`error`的话就把`error`返回出来

这次的命令行还是使用的[commander.js](https://github.com/tj/commander.js "commander.js")，想必大🔥儿们已经很熟悉了，就不再赘述

## 后记

这个小工具主要是实践了一下如何发请求，以及在node14版本以上使用`URLSearchParams`API，欢迎各位和我一起交流讨论\~
