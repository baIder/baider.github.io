---
title: 一个命令行todo工具
date: 2022-08-06 15:29:14
categories:
  - Node
tags:
  - Node
---

# \[Cli Todo] 一个命令行todo工具

## 前言

`bald3r-node-todo`是一个用node.js开发的，主要用于命令行的todo工具，主要使用了fs模块，目前已经发布至npm

本工具主要使用了面向接口的编程思想，并用jest进行单元测试

## 链接

[baIder/node-todo (github.com)](https://github.com/baIder/node-todo "baIder/node-todo (github.com)")

[bald3r-node-todo - npm (npmjs.com)](https://www.npmjs.com/package/bald3r-node-todo "bald3r-node-todo - npm (npmjs.com)")

## 使用演示

1.  首先使用`yarn`或`npm`安装`bald3r-node-todo`
    ```bash
    npm install bald3r-todo
    yarn global add bald3r-todo

    ```
2.  安装完成后就可以使用全局命令`t`来使用了

    ![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_PFI5q2iQdM.png)
3.  使用命令行添加一个待办`t add [taskName]`

    ![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_lXiFyGko6H.png)
4.  查看当前待办

    ![](https://img.bald3r.wang/img/20220806223935.png)
5.  二级菜单

    ![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_2le53n2lJK.png)

    ![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_JtaVVHAKR2.png)
6.  清空所有待办`t clear`

    ![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_E1-fjw9GOi.png)

## 实现过程

### 实现命令行参数

这里我使用了[commander库](https://github.com/tj/commander.js#readme "commander库")来实现参数功能

```javascript
program
  .command('add')
  .description('add a task')
  .action((...args) => {
    const words = args.slice(0, -1).join(' ')
    api.add(words).then(() => {
      console.log('The task has been successfully added')
    }, () => {
      console.log('Failed to add the task')
    })
  })

program
  .command('clear')
  .description('clear all tasks')
  .action(() => {
    api.clear().then(() => {
      console.log('All tasks have been successfully removed')
    }, () => {
      console.log('Failed to remove all the tasks')
    })
  })
```

commander默认会有两个参数，一个是node的路径，一个是当前文件的路径，因此我们判断参数的数量是否为2就可以判断用户是否传参

如果用户没有传参，则显示所有的待办项

```javascript
if (process.argv.length === 2) {
  api.showAll()
}
```

### 实现可以操作的命令行

这里我使用了[inquirer库](https://github.com/SBoudrias/Inquirer.js "inquirer库")来给命令行做了美化，实现可以用方向键和回车控制的UI界面

inquirer的使用非常简单，这里我展示二级菜单作为参考

```javascript
function askForAction(list, index) {
  const actions = {markAsUndone, markAsDone, changeTitle, removeTask}
  inquirer.prompt({
    type: 'list',
    name: 'action',
    message: 'What to do with the task?',
    choices: [
      {name: 'Exit', value: 'quit'},
      {name: 'Mark as Done', value: 'markAsDone'},
      {name: 'Mark as Undone', value: 'markAsUndone'},
      {name: 'Edit Title', value: 'changeTitle'},
      {name: 'Delete', value: 'removeTask'},
    ]
  }).then(answer2 => {
    const action = actions[answer2.action]
    action && action(list, index)
  })
}
```

这样便实现了下图的二级菜单

![](image/image_JtaVVHAKR2.png)

### 待办项保存在本地

使用node.js的`fs`模块来实现对文件的读写，这里涉及一个保存路径的问题，在本项目中，为了方便使用了`~`目录，所有数据保存在`~/.todo`中

获取`~`目录：

```javascript
const homedir = require('os').homedir()
const home = process.env.HOME || homedir
```

考虑到跨平台使用路径的表示方式不同，这里使用了node.js中的`path`模块：

```javascript
const p = require('path')
const dbPath = p.join(home, '.todo')
```

然后使用`fs`模块中的`fs.readFile()`和`fs.writeFile()`即可完成对数据的读写。这里需要注意这两个操作都是异步的，因此用到了`Promise`，这里的`{flag: 'a+'}`是表示读取文件，若不存在则创建一个：

```javascript
read(path = dbPath) {
  return new Promise((resolve, reject) => {
    fs.readFile(path, {flag: 'a+'}, (error, data) => {
      if (error) return reject(error)
      let list
      try {
        list = JSON.parse(data.toString())
      } catch (error2) {
        list = []
      }
      resolve(list)
    })
  })
}
```

## 后记

这是一个非常简单的小应用，如果你有任何的意见和建议，可以留言给我哦\~
