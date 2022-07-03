# CSS学习笔记

CSS-Cascading Style Sheets

目前使用最广泛的是CSS2.1，现代版本为CSS3，仍在分模块升级种。

## border调试法

不加border就不要写代码！

## 文档流 Normal Flow

文档流动方向，从左至右，从上至下。

### 流动方向

inline元素从左往右，到达最右边换行；

block元素从上往下，每个block元素独占一行；

inline-block元素从左往右。

### 宽度

inline元素宽度不可用width指定，为内部inline元素的和；

block元素宽度可用width指定；

inline-block元素兼具两者特点，我的理解是具有block属性但是像inline元素一样不必独占一行的元素。

### 高度

inline高度由line-height间接确定；

block可以用height指定；

inline-block可以用height指定。

### overflow 溢出

可设置是否显示滚动条。

overflow:auto | scroll | hidden | visible

### 脱离文档流

float 和 position:absolute/fixed 可使元素脱离文档流

## 盒模型


CSS盒模型将CSS中的元素整体视为一个盒子BOX，这个box由四个部分构成：

- margin
- border
- padding
- content

CSS盒模型分两种

- content-box
- border-box

### 区别

区别主要在于width（或height）包含的范围。

- content-box：`width`只控制`content`部分的`width`
- 
- border-box：`width`控制`border+padding+content`三部分的`width`

由区别可知，若未指定`border`和`padding`，则两种盒模型并无区别。


**通常情况下我们都会使用`border-box`因为该盒模型更好用。**

因为`margin`会合并，对元素大小的影响比较小，而`border`和`padding`的大小已经包含在`width`中，因此`border-box`能够更直观的控制元素的大小，同时更符合border调试法的使用习惯。

### margin合并

父子margin合并，兄弟margin合并

阻止合并的方法：

- 父子合并: padding/border
- 父子合并: overflow: hidden
- 父子合并: display: flex
- 兄弟合并: inline-block

## 布局

布局指二维平面的布局，注意与“定位”做区分。

### float布局

主要用于IE，现在用的很少

步骤：

1. 子元素加上float: left | right
2. 子元素加上width
3. 父元素的class中加入.clearfix

.clearfix代码

```HTML

.clearfix{
    content:"";
    display:block;
    clear:both;
}

```

IE 6/7 存在双倍margin bug，解决方法如下：

1. margin减半
2. display:inline-block

### flex布局

主流布局

需要记住的代码：

```HTML

display:flex
flex-direction: row | column
flex-wrap: wrap
justify-content: center | space-between
align-items: center

```

[flex - css tricks](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)

### grid布局

无敌强大面向未来，有兼容性问题

适合不规则布局

默认将页面分为了3行5列的表格，通过行线和竖线来定位区域


[grid - css tricks](https://css-tricks.com/snippets/css/complete-guide-grid/)



## 定位

垂直于屏幕

div的分层

- inline 元素
- float 元素
- 块级子元素
- border
- background

### position属性

```HTML

position: static        默认

position: relative      相对定位，float但不脱离文档流

position: absolute      绝对定位，基准是祖先里的非static

position: fixed         固定定位，基准是viewport

position: sticky        会粘在viewport顶部

```

写了absolute要补relative

写了absolute或fixed要补top和left

sticky兼容性很差，比如在缩放时会有bug

### z-index

```HTML
z-index: auto       不创建新层叠上下文
z-index: 一个数字

```

```HTML

.demo{
    position: fixed;
    left: 50%;
    top: 50%;
    transform: translate(-50%，-50%)；
}

<!-- 绝对定位元素居中 -->

```

### 层叠上下文

每个层叠上下文是一个新的作用域，该作用域内的z-index与外界无关，同一个作用域内的z-index才能比较

z-index / flex / opacity / transform 会创建一个新的层叠上下文

## CSS动画

### 浏览器渲染过程

1. 根据HTML构建HTML树 (DOM)
2. 根据CSS构建CSS树（CSSOM）
3. 将两棵树合并成一颗渲染树（render tree)
4. Layout
5. Paint
6. Composite

### 三种更新方式

1. JS/CSS > Style > Layout > Paint > Composite
2. JS/CSS > Style > Paint > Composite
3. JS/CSS > Style > Composite

div.remove() 触发第一种更新方式

改变背景颜色 触发第二种更新方式

transform 触发第三种更新方式

[CSS动画优化](https://developers.google.com/web/fundamentals/performance/rendering/stick-to-compositor-only-properties-and-manage-layer-count)

### transform

[transform语法](https://developer.mozilla.org/zh-CN/docs/Web/CSS/transform)

常用代码：

- transform: translate ()
- transform: scale ()
- transform: rotate ()
- transform: skew ()

配合transition使用：
`transition: all 1s ease 0.5s;` 


### transition

给动画补充中间帧

transition:属性名 时长 过渡方式 延迟

### animation

步骤：

1. 声明关键帧
2. 添加动画

```HTML
#demo.start{
  animation: xxx 1.5s;
}

@keyframes xxx {
  0% {
    transform: none;
  }
  66.66%{
    transform: translateX(200px);
  }
  100%{
    transform: translateX(200px) translateY(100px);
  }
}

```

```HTML
/* @keyframes duration | timing-function | delay |
   iteration-count | direction | fill-mode | play-state | name */
animation: 3s ease-in 1s 2 reverse both paused slidein;

animation: 时长 | 过渡方式 | 延迟 | 次数 | 方向 | 填充模式 | 是否暂停 | 动画名；

```

