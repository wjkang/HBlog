title: CSS中的BFC，外边距折叠，清除浮动
author: 若邪
tags:
  - CSS
categories: []
date: 2017-07-13 15:43:00
---
### BFC是什么？
BFC（Block Formatting Context）直译为“***块级格式化范围*** ”。

>是 W3C CSS 2.1 规范中的一个概念，它决定了元素如何对其内容进行定位，以及与其他元素的关系和相互作用。当涉及到可视化布局的时候，Block Formatting Context提供了一个环境，HTML元素在这个环境中按照一定规则进行布局。一个环境中的元素不会影响到其它环境中的布局。

### BFC 的特征
在BFC中，盒子从顶端开始垂直地一个接一个地排列，两个盒子之间的垂直的间隙是由他们的margin 值所决定的。在一个BFC中，两个相邻的块级盒子的垂直外边距会产生折叠。

在BFC中，每一个盒子的左外边缘（margin-left）会触碰到容器的左边缘(border-left)（对于从右到左的格式来说，则触碰到右边缘）。

BFC中的元素的布局是不受外界的影响（我们往往利用这个特性来消除浮动元素对其非浮动的兄弟元素和其子元素带来的影响。）并且在一个BFC中，块盒与行盒（行盒由一行中所有的内联元素所组成）都会垂直的沿着其父元素的边框排列。

计算BFC的高度时，浮动元素也参与计算（可用于解决浮动造成的高度塌陷问题）

BFC的区域不会与float box重叠(解决浮动元素文字环绕问题)

### 触发BFC
- float 的值不为none
- position 的值不为static或者relative
- display的值为 table-cell, table-caption, inline-block, flex, 或者 inline-flex中的其中一个
- overflow的值不为visible
满足上述条件中至少一项，即可触发 BFC

### 使用场景

#### 1、外边距折叠问题

折叠的条件：两个元素的 margin 必须是 相邻 的；两个 margin 是邻接的必须满足以下条件：
- 必须是处于常规文档流（非float和绝对定位）的块级盒子,并且处于同一个 BFC 当中。
- 没有inline盒子，没有空隙，没有 padding 和 border 将他们分隔开。
- 都属于垂直方向上相邻的外边距。
如下：
```
<div clss="container">
    <div class="f-block">
    </div>
    <div class="s-block">
    </div>
</div>
```
```
.f-block
{
  width:200px;
  height:200px;
  background-color:silver;
  margin:10px;
}
.s-block
{
  width:200px;
  height:200px;
  background-color:silver;
  margin:10px;
}
```
![](http://upload-images.jianshu.io/upload_images/2125695-622c3f6fc1d8b669.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

解决方法：
破坏产生折叠的条件即可：
1、给元素添加浮动或者绝对定位（会影响整体布局，改动大）
2、将元素改为行内元素（会影响整体布局，改动大）
3、元素间插入一个高度大于0的div
4、使元素不在同一BFC中

```
  <!--元素间插入一个高度大于0的div-->
  <div class="f-block">
  </div>
  <div style="height:0.01px"></div>
  <div class="s-block">
  </div>
```

![元素间插入一个高度大于0的div](http://upload-images.jianshu.io/upload_images/2125695-abe89b14510dc229.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
<div class="f-block"></div>
<div>
  <div class="s-block"></div>
</div>
```
f-block,s-block以及s-block外层div已经还是处在相同BFC中，所以还是会产生外边距折叠。这里产生的折叠比较复杂：
首先是s-block和外层div的外边距折叠，然后是合并后的折叠外边距再与f-block产生外边距折叠。
下面解决父子元素的外边距折叠问题：

**给父元素添加padding 或 border（破坏外边距折叠条件）**
```
<div class="f-block"></div>
<div style="background-color:red;border:1px solid black">
  <div class="s-block"></div>
</div>
```

![](http://upload-images.jianshu.io/upload_images/2125695-d1892c0eba36f877.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


**给父元素创建新的块级格式化上下文（创建了新的块级格式化上下文的块元素，不与它的子元素发生margin 折叠）**
```
<div class="f-block"></div>
<div style="background-color:red;overflow:hidden">
  <div class="s-block"></div>
</div>
```
![](http://upload-images.jianshu.io/upload_images/2125695-6050f96cc1f7c74d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 2、清除浮动
```
<div class="container">
  <div class="item"></div>
  <div class="item"></div>
  <div class="item"></div>
  <div class="item"></div>
  <div class="item"></div>
</div>
```
```
.container
{
  width:550px;
  border:5px solid red;
}
.item
{
  width:100px;
  height:100px;
  float:left;
  background-color:silver;
  margin:5px;
}
```
对于上面的代码，我们希望得到这样的结果：
![](http://upload-images.jianshu.io/upload_images/2125695-5f14985076b9a5a7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

但结果却是这样的：
![](http://upload-images.jianshu.io/upload_images/2125695-86de3ce2fa4746f2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

原因是因为浮动元素脱离文档流，container的高度无法被撑开。

清除浮动的方法及原理：

**clear属性**：
clear属性的意义就是禁止特定方向上存在浮动元素，例如clear: left就是不允许元素左方存在文档流之前浮动元素（注意，这里左边如果存在文档流之后的浮动元素是无法清理的）
根据CSS权威指南，具体的实现原理是通过引入清除区域，这个相当于加了一块看不见的框把定义clear属性的元素向下挤，直到元素指定方向刚好没有浮动元素，这样看起来包含框就把浮动框给包含了，实际上浮动框脱离文本流的性质没变，它们依然是“浮”在上面的
用法：
1、容器内结尾处加空div标签 clear:both 
2、容器定义伪类:after 和 zoom(IE8以上和非IE浏览器才支持:after，原理和方法2有点类似，zoom(IE转有属性)可解决ie6,ie7浮动问题)
```
.container:after{display:block;clear:both;content:"";visibility:hidden;height:0}
.container{zoom:1}
```

**容器形成新BFC**：
计算BFC的高度时，浮动元素也参与计算

```
.container
{
  width:550px;
  border:5px solid red;
  overfow:hidden;
}
```
或者使容器自身浮动也可以

消除浮动引起的文字环绕效果：
```
<div class="box">
    <div class="img">image</div>
    <div class="info">信息信息信息信息信息信息信息信息信息信息信息信信息信息信息信息信息信息信息信息信息信息信息信信息信息信息信息信息信息信息信息信息信息信息信信息信息信息信息信息信息信息信息信息信息信息信</div>
</div>
```
```
.box {width:210px;border: 1px solid #000;float: left;} 
.img {width: 100px;height: 100px;background: #696;float: left;} 
.info {background: #ccc;color: #fff;}
```

![](http://upload-images.jianshu.io/upload_images/2125695-6527144ca5ae7997.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 带有浮动属性的元素会脱离标准流进行排列布局，脱离标准流后的元素就不和块元素相处在同一个流不居中，使得带有浮动属性的元素漂浮在正常块元素上面。但是**浮动的块虽然脱离了正常的文档流，但是依然占据正常文档流的文本空间**。于是在其后面写的文本并不会被浮动元素所覆盖而是继续水平排列超出换行。
标准流中块元素每个各占一行。内联元素则是水平排列，直到一行排列不下进行换行操作。因为使用了float的元素占据文本空间，使得后面的文本以除了浮动元素之外的空间为排列基准，形成了文本环绕的效果。

清除环绕效果就是使info形成一个新的BFC（BFC的区域不会与float box重叠），添加overflow:hidden即可。
注意：给info绝对定位也会形成一个新的BFC，但是info会按照绝对定位的规则进行布局，会与img重叠。

参考文章
[深入理解BFC和Margin Collapse ](http://www.w3cplus.com/css/understanding-bfc-and-margin-collapse.html)
[细说CSS中的BFC](https://juejin.im/post/583bb606a22b9d006c141286)