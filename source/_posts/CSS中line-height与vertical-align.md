title: CSS中line-height与vertical-align
author: 若邪
tags:
  - line-height
  - vertical-align
categories:
  - 前端
date: 2017-03-28
---
>参考文章：
>[深入了解CSS的line-height属性](http://mp.weixin.qq.com/s?__biz=MzAwNzAzMDY1NQ==&mid=206445305&idx=1&sn=899beb2d7e98194c000f9818c79f7778&scene=0#wechat_redirect)
>[Vertical-Align: 你需要知道的所有事【译】](http://mxd.tencent.com/vertical-align)
>[Vertical-Align: All You Need To Know](http://christopheraue.net/2014/03/05/vertical-align/)


### 1、什么是行间距或者行高（line-height）
line-height是指文本行基线间的垂直距离。

#### 1.1、顶线，中线，基线，底线
![](http://upload-images.jianshu.io/upload_images/2125695-757a5ead60960914.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

从上到下分别是顶线，中线，基线，底线。vertical-align的四个属性top,middle,baseline,bottom就是与这四条线有关。

#### 1.2、行高，行距，半行距
- 行高是指上下文本行基线间的垂直距离。（上图中两条红线间的垂直距离）
- 行距是指一行底线到下一行顶线的垂直距离。（第一条粉线和第二条绿线间的垂直距离）
- 半行距就是行距/2。(图中可以看出，半行距=(行高-字体size)/2 )
![](http://upload-images.jianshu.io/upload_images/2125695-0d6db0a0be6aea55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 1.3、内容区，行内框，行框

![](http://upload-images.jianshu.io/upload_images/2125695-c2700d187c56e2ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- 内容区：顶线和底线包裹的区域（字体的size）
- 行内框：在没有其他因素影响的时候（padding等），行内框等于内容区。而设定行高时行内框高度不变，半行距分别增加/减少到内容区的上下两边（深蓝色区域）行框（line box）。（字体size不变，修改行高就是修改行距）
- 行框：行框高度等于本行内所有元素中行内框最大的值（以行高值最大的行内框为基准，其他行内框采用自己的对齐方式向基准对齐，最终计算行框的高度），当有多行内容时，每行都会有自己的行框。

#### 1.4、line-height的设置
**百分比方式设置**
```
<body>
  121212
  <p>121212</p>
</body>
```
```
body{
  font-size:16px;
  line-height:120%;
}
p{
  font-size:32px;
}
```
line-height的百分比（120%）和body的字体大小（16px），被用来计算（16*120=19.2），这个值会被层叠下去的元素所继承。
![](http://upload-images.jianshu.io/upload_images/2125695-64f8ac452bd0518e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


![](http://upload-images.jianshu.io/upload_images/2125695-78b4087a3cd4be60.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**补充**
```
p{
  font-size:32px;
  line-height:60px;
  padding:10px
}
```
最终盒模型

![](http://upload-images.jianshu.io/upload_images/2125695-0c2e089a48a24e41.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/2125695-433cdef3b48b6d29.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

盒模型中，内容（不是上文说的内容区，上文的内容区是顶线与底线间的区域）的高度等于line-height的值。为什么会有margin？浏览器默认P的上下margin是1em,设置了P的font-size是32px,所以1em=32px。上下margin就是32px。

**长度方式（px）设置**
```
<body>
  121212
  <p>121212</p>
</body>
```
```
body{
  font-size:16px;
  line-height:20px;
}
p{
  font-size:32px;
}
```
![](http://upload-images.jianshu.io/upload_images/2125695-10466171680bb1e4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/2125695-f96f2fa3cd805769.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**值normal**
```
<body>
  121212
  <p>121212</p>
</body>
```
```
body{
  font-size:16px;
  line-height:normal;
}
p{
  font-size:32px;
}
```

![body](http://upload-images.jianshu.io/upload_images/2125695-07ca5eb22b4f162d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

body的line的line-height是22px,所以normal等于1.375

![](http://upload-images.jianshu.io/upload_images/2125695-6f20ee57b87ffe34.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

p的line-height：32px*1.375=44px（normal并不是精确的等于1.375）

![](http://upload-images.jianshu.io/upload_images/2125695-fa34a0ff48157fea.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**纯数字**
就是将normal改为一个想要的准确数字。

#### 1.5、各种BOX
```
<body>
  <p>这个<em>强调</em> 元素为行内元素</p>
</body>
```
```
body{
  font-size:16px;
  line-height:1.5;
}
p{
  font-size:32px;
  padding:10px;
}
```

![](http://upload-images.jianshu.io/upload_images/2125695-6da3ca94f2274bd4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**containing box**
p就是一个containing box，包含了其他boxs。

**inline box**
在段落内，有一系列的inline box,inline box不会让内容成块显示，而是排成一行。“强调”是一种inline box,“这个”，“元素为行内元素”为一种匿名inline box。

**line box**
多个inline box组成line box，多个line box组成containing box。

**Content Area**
Content Area是围绕着文字的一种看不见的box,高度取决与font-size

**inline box与line-height**
font-size:32px，line-height:48px，行间距=48px-32px=16px，半行间距=8px。
半行间距会用在Content Area的顶部和底部。

![](http://upload-images.jianshu.io/upload_images/2125695-54c422b0a06e7682.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
这里inline box的高度就是line-height。inline box包着Content Area

但是，当line-height小于font-size。line box的高度还是line-height,所以line-box的高度小于Content Area的高度，Content Area会溢出line-box。

![](http://upload-images.jianshu.io/upload_images/2125695-3c057a9446434303.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**inline box 与line box**
line box的高度取决于他内部最高的inline box。

![](http://upload-images.jianshu.io/upload_images/2125695-bc4f96a4733b1fc5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/2125695-fbe2f27a7b3c5d2b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/2125695-3f348ef9d532567b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![](http://upload-images.jianshu.io/upload_images/2125695-d7d9db120455ff05.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 2、vertical-align
vertical-align是用来对齐内联级元素的。设置为以下display属性的元素，它们都被认为属于内联级元素。**inline**、**inline-block** or **inline-table **(本文中不涉及此种情况)：

inline内联元素基本上是包裹文本的标签。

inline-block内联块元素则如它们的名字所示：拥有内联特性的块元素。他们可以有width和height（可能是由自己的内容定义），以及padding、border和margin。

内联级元素彼此紧挨着放在一行中。一旦有更多的元素被放置到当前行中，一个新的行将会在它下面创建。所有这些行有所谓的“行框”，行框中包含所有的内容。不同大小的内容意味着不同高度的行框。在下面的插图中，行框的顶部和底部都是用红线表示的。
![](http://upload-images.jianshu.io/upload_images/2125695-7aab0dba5bc20d39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在行框中，元素的vertical-align属性是负责垂直对齐的。那么，到底元素垂直对齐的参照物是什么？

**参照物：父元素的基线和外边缘**
看看元素的基线和行框的外观：
**inline元素**：

![](http://upload-images.jianshu.io/upload_images/2125695-08b7390e62d919fd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

三行并排的文本。行框的顶部和底部边缘用红线表示，字体的高度由绿线表示，基线由一条蓝线表示。在左边，有一个line-height设置为与字体font-size大小相同高度的文本，绿线和红线重叠在一条线上。在中间，line-height是字体的两倍大。在右边，line-height是字体大小的一半大。

内联元素（display:inline）的外边缘与其行高的顶部和底部边缘对齐，行高可以小于字体的高度。所以，行框就是上面的图中的红线。

内联元素的基线是字符放置的位置线（字母x底部所在的水平线），即图中的蓝线。粗略地说，基线是在字体1/2高度的下面的某个地方。

**inline-block元素**：
inline-block因为已经有宽和高，可能存在多行，每行都有自己的基线和行框，所以会比较特殊。

![](http://upload-images.jianshu.io/upload_images/2125695-924628ed8ffdeee1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

上图中，最外层是div，里面分别是三个inline-block的span，黄色为border，绿色为padding，蓝色为content area（一个span，其中有一个字母“C”）。左边的inline-block的span的内容（span）是一个正常文档流元素。中间的inline-block的span还额外加了overflow: hidden。右边的inline-block的span包含一个流外的span(但内容区域有一个高度)(译者注：流内的元素必须是普通文档流（normal flow）中的元素，流外的元素必须是浮动或绝对定位的元素以及根元素。)。蓝线为每个inline-block的span的基线。内联块元素的外边缘是其margin框的顶部和底部边缘，即图中的红线。

内联块元素（上图三个inline-block的span）的基线取决它包含的内容是否在文档流中：
- 在流内内容的情况下，内联块元素的基线是正常流中最后一个内容元素的基线（左边的例子）。对于这最后一个元素，它的基线是根据它自己的规则找到的。

```
<div class="demo1">
  x<span>
    x<span style="display:inline-block;height:30px;width:100px;background-color:blue">x</span>
    x
  </span>
</div>
```
```
.demo1 span{
  display:inline-block;
  background-color:silver;
  height:90px;

}
```

![](http://upload-images.jianshu.io/upload_images/2125695-e1f47b4cd9e6e5d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
灰色背景的元素内部有三个子元素，两个“x”,一个span。元素的基线就是最后一个正常流元素（“x”）的基线。
修改元素的长度，使其内容出现多行：

![](http://upload-images.jianshu.io/upload_images/2125695-62c45b8a73e166db.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

最外面的X怎么也跟着移动了？这涉及行框基线的移动，下文细说。
- 在流内内容但内联块元素有overflow:hidden属性的情况下，基线是内联块元素margin框的底部边缘（例如在中间的图）。
修改上面的例子样式：
```
.demo1>span{
  display:inline-block;
  background-color:silver;
  height:90px;
  width:100px;
  margin:10px;
  overflow:hidden;
}
```

![](http://upload-images.jianshu.io/upload_images/2125695-97fa3f9c028cfce8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
通过最外面的x大致知道行框的基线位置，就是内联块元素的下外边距的地方，也是内联块元素元素的基线位置。
**一开始此处有疑惑**：内联块元素元素的基线跑到了下外边距处，那么元素里面的内容不应该以这条基线做定位吗？群里问了大佬，内联块元素已经设了宽高，可能有多行（即使只有一行），每行有各自的行框，然后又根据规则定位了，跟内联块元素的基线已经没有关系。

- 在流外内容的情况下，基线是内联块元素margin框的底部边缘（例如在右边）。

```
<div class="demo1">
  x<span>
  <span style="display:inline-block;height:30px;width:100px;background-color:blue;">x</span>
  </span>
</div>
```
```
.demo1>span{
  display:inline-block;
  background-color:silver;
  height:90px;
  width:100px;
}
```

![](http://upload-images.jianshu.io/upload_images/2125695-1bf79648253f0178.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

加上浮动
```
<div class="demo1">
  x<span>
  <span style="display:inline-block;height:30px;width:100px;background-color:blue;float:left">x</span>
  </span>
</div>
```

![](http://upload-images.jianshu.io/upload_images/2125695-e1340e0337c09cb1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**行框的基线是可变的**
当使用vertical-align时，基线放置在哪里可能是最令人疑惑的部分。它需要满足vertical-align的值和行框的高度等所有条件。基线的位置犹如是方程中的一个自由参数。

行框的基线是看不见的，但你可以使它很容易看到。只要在文本行的开头添加一个字符，像我增加了一个“X”的字母。如果这个字符不以任何方式对齐，它将默认地坐在基线上。

![](http://upload-images.jianshu.io/upload_images/2125695-17a321a13fb1de4e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

围绕着行框的基线的部分（绿线），我们可以称其为文本框。文本框可以简单地被认为是行框内的内联元素，没有任何对齐。文本框的高度等于它的父元素的字体大小。因此，文本框只围住了行框内的无格式文本。由于这个文本框是绑在基线上的，当基线移动时它将移动。（注：此文本框在W3C规范中称为“strut（支柱）”）

**vertical-align的值**
**1）将元素的基线，参照父元素的基线对齐**


![](http://upload-images.jianshu.io/upload_images/2125695-be511377e0cbfe01.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

baseline：元素的基线与父元素的基线对齐。

sub：元素的基线偏移到父元素的基线之下。

sup：元素的基线偏移到父元素的基线之上。

<percentage>：元素的基线相对于父元素的基线偏移了一个百分比（该百分比是对比元素自身的line-height计算得出）。

<length>：元素的基线相对于父元素的基线偏移了一个绝对长度。

**2）将元素的中心点，参照父元素的基线对齐**

![](http://upload-images.jianshu.io/upload_images/2125695-7e62e2f007fae25b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

middle：将元素的顶部和底部之间的中心点，对齐父元素的基线之上x-height的1/2之处（x-height为字母x的字符高度）。

**3）将元素的外边缘，参照父元素的文本框对齐**

![](http://upload-images.jianshu.io/upload_images/2125695-ad50fa22b3971468.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

text-top：将元素的顶部边缘，对齐到父元素的文本框的顶部边缘。

text-bottom：将元素的底部边缘，对齐到父元素的文本框的底部边缘。

**4）将元素的外边缘，参照父元素行框的外边缘对齐**

![](http://upload-images.jianshu.io/upload_images/2125695-22702eed5b65c234.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

top：元素的顶部边缘对齐到父元素的顶部边缘。

bottom：元素的底部边缘对齐到父元素的底部边缘。

**基线的移动**
如果一行中有一个高个的元素占据了整行的高度，那么vertical-align对它没有影响。它的顶部和底部没有空间让它移动。为了满足行框基线的对齐方式，行框的基线必须移动。矮个元素设置了vertical-align: baseline。在左边，高个元素设置了vertical-align: text-bottom。在右边，高个元素设置了vertical-align: text-top。你可以看到右边的基线跳起来了。

![](http://upload-images.jianshu.io/upload_images/2125695-c1814fe05aab774b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

（左）将两个元素放在一行中并设置vertical-align ，它们会使得行框的基线移动到符合它俩的对齐规则之处，然后行框的高度也会随之调整。（中）添加第三个元素，不超越行框的边缘，既不影响行框的高度，也不影响基线的位置。（右）添加第三个元素，如果它超出了行框的边缘，行框的高度和基线调整。在这种情况下，我们的前两个元素也会跟着发生变化。

![](http://upload-images.jianshu.io/upload_images/2125695-2116d86a4125a740.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**内联级元素底部的小间隙**

![](http://upload-images.jianshu.io/upload_images/2125695-09bed4e30dbca377.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

列表项坐在基线上。下面的一点空间，是文本的基线以下预留的depth（在W3C规范中，一个字体的基线以上称为characteristic height，基线以下称为depth）。想要去掉这个depth空隙，有解决的办法吗？只要移动基线的位置就可以，例如通过设置列表项目vertical-align: middle

**水平垂直居中**

```
<div class="box">
        <div class="content">
          自适应垂直居中
        </div>
</div>
```
```
html{
  height:100%;
}
body{
   height: 100%;
   width: 100%;
}
.box{
   display:inline-block;
   text-align: center;
   width:50%;
   height:50%;
   background-color:#e1e3cd;
   overflow:hidden;
}
.box:after{
    content:"";
    display:inline-block;
    height:100%;
    vertical-align:middle;
}
.content{
    vertical-align:middle;
    background-color:silver;
    display: inline-block;
    width: 50%;
    height:50%;
}
```

![](http://upload-images.jianshu.io/upload_images/2125695-0ebe9d2f22f572c6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

要将content水平垂直居中定位在box里，利用vertical-align是其中一种方法。原理是：vertical-align：middle（将元素的顶部和底部之间的中心点，对齐父元素的基线之上x-height的1/2之处（x-height为字母x的字符高度）。），content肯定是要垂直居中的，那只能修改行框的基线位置（**注意：不是修改box的基线，box具有宽高，它里面的内容可能会有多行，每行有各自的行框，box的基线已经不会影响内容的布局，但是box的基线还是会受里面内容的影响（内联块元素的基线是正常流中最后一个内容元素的基线）**），使其位于box的垂直中心位置。修改行框的基线，只要在box内加一个高度为100%的空元素，然后设置vertical-align:middle，添加的元素已经占满整个行框高度，而只要移动行框的基线，就可以满足定位规则，所以行框的基线就被移动到box垂直中心位置。content再按规则对齐到行框基线上就可以了。