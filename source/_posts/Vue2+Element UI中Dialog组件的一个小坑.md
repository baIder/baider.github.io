---
title: Vue2+Element UI中Dialog组件的一个小坑
date: 2023-02-10 15:22:41
tags:
  - Vue2
  - Element UI
---

# Vue2+Element UI中Dialog组件的一个小坑

### 太长不看版

Vue2 + Element UI中的Dialog组件在使用过程中会有一个小bug。

根据文档，将`destroy-on-close`设置为`true`后，Dialog将会在关闭后销毁其中的组件，但是实际上Dialog会在销毁组件后再次挂载销毁的组件，因此可能会影响我们对组件行为的预期。

例如Dialog中有一个组件A，A在挂载时向Vuex的store中存入了变量a，值为true，A在销毁时将变量a设置为了false，那么由于Dialog会在关闭后销毁并重新挂载组件A，Vuex的store中的a实际上还是true。

### Element UI中的Dialog组件

[组件 | Element](https://element.eleme.cn/#/zh-CN/component/dialog "组件 | Element")

可以打开一个对话框，对话框中可以嵌入其他的组件，当打开对话框时，该对话框中的组件会mount，但是关闭对话框时，该组件不会销毁

```vue
// 对话框
    <el-dialog
        title="提示"
        :visible.sync="dialogVisible"
        width="30%"
        :destroy-on-close="true"
        :before-close="handleClose">

      <ComponentA/>

      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="dialogVisible = false">确 定</el-button>
      </span>
    </el-dialog>
 
// ComponentA
<template>
  <div>ComponentStatus:{{status}}</div>
</template>

<script>
export default {
  name: "ComponentA",
  data(){
    return {
      status:'',
    }
  },
  mounted() {
    this.status='mounted'
    console.log('mounted once')
  },
  beforeDestroy() {
    this.status='unmounted'
    console.log('unmounted once')
  }
}
</script>
```

点击打开Dialog后，ComponentA正常挂载，关闭对话框，ComponentA没有销毁

![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_BE91vL-hBn.png)

![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_9PgJ6-rK-F.png)

### destroy-on-close

根据文档，将`destroy-on-close`设置为`true`可以在关闭对话框时销毁其中元素

```vue
    <el-dialog
        title="提示"
        :visible.sync="dialogVisible"
        width="70%"
        :destroy-on-close="true"   //这里
        :before-close="handleClose">

      <ComponentA/>

      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false">取 消</el-button>
        <el-button type="primary" @click="dialogVisible = false">确 定</el-button>
      </span>
    </el-dialog>
```

![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_Jm8wyhkFTQ.png)

![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_l_7fNTsy9f.png)

> 📌但实际上组件会在销毁后再次被挂载，其中就会引发Bug

### 曲线救国方案：v-if

实际上就是在对话框打开的时候将组件的v-if置为true，在对话框关闭时将v-if置为false

## Vue3+Element Plus正常

[Dialog 对话框 | Element Plus (gitee.io)](https://element-plus.gitee.io/zh-CN/component/dialog.html "Dialog 对话框 | Element Plus (gitee.io)")

![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_9vGUcm1enP.png)

![](https://balder-wang-images.oss-cn-shanghai.aliyuncs.com/img/image_0-M-YK-vI7.png)

对话框关闭后正常销毁组件，没有重新挂载
