title: CSS实现Tab布局
author: Jayce
tags:
  - Tab布局
  - ''
categories:
  - CSS
date: 2017-07-12 16:21:00
---
### 一、布局方式
#### 1、内容与tab分离
```
<div class="container">
   <div class="tab-content">
     <div id="item1" class="item">内容1</div>
     <div id="item2" class="item">内容2</div>
     <div id="item3" class="item">内容3</div>
     <div id="item4" class="item">内容4</div>
   </div>
   <div class="tab-control">
     <ul>
        <li><a href="#item1">内容1</a></li>
        <li><a href="#item2">内容2</a></li>
        <li><a href="#item3">内容3</a></li>
        <li><a href="#item4">内容4</a></li>
     </ul>
   </div>
</div>
```
```
ul,li{
  margin:0;
  padding:0;
  list-style:none;
}
.container{
  width:400px;
  height:300px;
  background-color:silver;
}
.tab-content{
  width:100%;
  height:80%;
  overflow:hidden;
}
.tab-content .item{
  width:100%;
  height:100%;
}
.tab-control{
  width:100%;
  height:20%;
}
.tab-control ul{
  height:100%;
}
.tab-control li{
  width:25%;
  height:100%;
  float:left;
  border:1px solid silver;
  box-sizing:border-box;
  background-color:white;
  cursor: pointer;
}
.tab-control li:hover{
  background-color:#7b7474
}
.tab-control a{
  display:inline-block;
  width:100%;
  height:100%;
  line-height:100%;
  text-align:center;
  text-decoration: none;
}
.tab-control a::after{
  content:"";
  display:inline-block;
  height:100%;
  vertical-align:middle;
}
.tab-content .item:target{
  background:yellow;
}
```

![](http://upload-images.jianshu.io/upload_images/2125695-bc4ca39b535f7aa3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



#### 2、内容与tab一体
```
<div class="container">
   <ul>
     <li class="item active">
       <p class="title">1</p>
       <p class="content">1</p>
     </li>
     <li class="item">
       <p class="title">2</p>
       <p class="content ml1">2</p>
     </li>
     <li class="item">
       <p class="title">3</p>
       <p class="content ml2">3</p>
     </li>
     <li class="item">
       <p class="title">4</p>
       <p class="content ml3">4</p>
     </li>
   </ul>
</div>
```
```
ul,li,p{
  margin:0;
  padding:0;
  list-style:none;
}
.container{
  width:400px;
  height:300px;
  background-color:silver;
  border:1px solid silver;
}
.container ul{
  width:100%;
  height:100%;
  overflow:hidden;
}
.container .item{
  float:left;
  width:25%;
  height:100%;
  background-color:white;
}
.container .item .title{
  line-height:40px;
  border:1px solid silver;
  box-sizing:border-box;
  text-align:center;
  cursor:pointer;
}
.container .item .content{
  width:400%;
  height:100%;
  background-color:yellow;
}
.ml1{
  margin-left:-100%;
}
.ml2{
  margin-left:-200%;
}
.ml3{
  margin-left:-300%;
}
.active{
  position:relative;
  z-index:1
}
.container .item:hover{
  position:relative;
  z-index:1
}
.container .item:hover .title{
  border-bottom:none;
  background-color:yellow;
}
```
![](http://upload-images.jianshu.io/upload_images/2125695-9d4160c5eda4d4c8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

利用负margin，将内容区对齐，然后内容去添加背景色，避免不同tab对应的区域透视重叠。

### 二、CSS实现交互
#### 1、锚点实现（target）
**(1)针对布局一**：item从上往下排列，父元素tab-content加上overflow:hidden。利用锚点，点击不同a标签的时候，具有对应ID的item会切换到tab-content的视图中，然后利用hover给tab按钮加上切换样式。
```
<div class="container">
   <div class="tab-content">
     <div id="item1" class="item">内容1</div>
     <div id="item2" class="item">内容2</div>
     <div id="item3" class="item">内容3</div>
     <div id="item4" class="item">内容4</div>
   </div>
   <div class="tab-control">
     <ul>
        <li><a href="#item1">内容1</a></li>
        <li><a href="#item2">内容2</a></li>
        <li><a href="#item3">内容3</a></li>
        <li><a href="#item4">内容4</a></li>
     </ul>
   </div>
</div>
```
```
ul,li{
  margin:0;
  padding:0;
  list-style:none;
}
.container{
  width:400px;
  height:300px;
  background-color:silver;
}
.tab-content{
  width:100%;
  height:80%;
  overflow:hidden;
}
.tab-content .item{
  width:100%;
  height:100%;
}
.tab-control{
  width:100%;
  height:20%;
}
.tab-control ul{
  height:100%;
}
.tab-control li{
  width:25%;
  height:100%;
  float:left;
  border:1px solid silver;
  box-sizing:border-box;
  background-color:white;
  cursor: pointer;
}
.tab-control li:hover{
  background-color:#7b7474
}
.tab-control a{
  display:inline-block;
  width:100%;
  height:100%;
  line-height:100%;
  text-align:center;
  text-decoration: none;
}
.tab-control a::after{
  content:"";
  display:inline-block;
  height:100%;
  vertical-align:middle;
}
```

上述方法只是利用了锚点切换，没有使用：target。修改CSS

```
ul,li{
  margin:0;
  padding:0;
  list-style:none;
}
.container{
  width:400px;
  height:300px;
  background-color:silver;
}
.tab-content{
  position:relative;
  width:100%;
  height:80%;
  overflow:hidden;
}
.tab-content .item{
  position:absolute;
  left:0;
  top:0;
  width:100%;
  height:100%;
}
.tab-control{
  width:100%;
  height:20%;
}
.tab-control ul{
  height:100%;
}
.tab-control li{
  width:25%;
  height:100%;
  float:left;
  border:1px solid silver;
  box-sizing:border-box;
  background-color:white;
  cursor: pointer;
}
.tab-control li:hover{
  background-color:#7b7474
}
.tab-control a{
  display:inline-block;
  width:100%;
  height:100%;
  line-height:100%;
  text-align:center;
  text-decoration: none;
}
.tab-control a::after{
  content:"";
  display:inline-block;
  height:100%;
  vertical-align:middle;
}

.tab-content .item:target{
  z-index:1;
  background-color:yellow;
}

```
item使用绝对定位，然后使用:target修改元素z-index达到切换效果（其实也可以通过控制元素的display来达到切换效果）

**（2）针对布局二：**
```
<div class="container">
   <ul>
     <li class="item active" id="item1">
       <p class="title"><a href="#item1">1</a></p>
       <p class="content">1</p>
     </li>
     <li class="item" id="item2">
       <p class="title"><a href="#item2">2</a></p>
       <p class="content ml1">2</p>
     </li>
     <li class="item" id="item3">
       <p class="title"><a href="#item3">3</a></p>
       <p class="content ml2">3</p>
     </li>
     <li class="item" id="item4">
       <p class="title"><a href="#item4">4</a></p>
       <p class="content ml3">4</p>
     </li>
   </ul>
</div>
```
```
ul,
li,
p {
  margin: 0;
  padding: 0;
  list-style: none;
}

.container {
  width: 400px;
  height: 300px;
  background-color: silver;
  border: 1px solid silver;
}

.container ul {
  width: 100%;
  height: 100%;
  overflow: hidden;
}

.container .item {
  float: left;
  width: 25%;
  height: 100%;
  background-color: white;
}

.container .item .title {
  line-height: 40px;
  border: 1px solid silver;
  box-sizing: border-box;
  text-align: center;
  cursor: pointer;
}
.container .item a {
  display:inline-block;
  width:100%;
  height:100%;
  text-decoration: none;
}

.container .item .content {
  width: 400%;
  height: 100%;
  background-color: yellow;
}

.ml1 {
  margin-left: -100%;
}

.ml2 {
  margin-left: -200%;
}

.ml3 {
  margin-left: -300%;
}

.active {
  position: relative;
  z-index: 1
}

.container .item:target {
  position: relative;
  z-index: 1
}

.container .item:target .title {
  border-bottom: none;
  background-color: yellow;
}
```

#### 2、hover实现
**（1）针对布局一：**
无法简单的通过CSS实现

**（2）针对布局二：**
```
<div class="container">
   <ul>
     <li class="item active">
       <p class="title">1</p>
       <p class="content">1</p>
     </li>
     <li class="item">
       <p class="title">2</p>
       <p class="content ml1">2</p>
     </li>
     <li class="item">
       <p class="title">3</p>
       <p class="content ml2">3</p>
     </li>
     <li class="item">
       <p class="title">4</p>
       <p class="content ml3">4</p>
     </li>
   </ul>
</div>
```
```
ul,li,p{
  margin:0;
  padding:0;
  list-style:none;
}
.container{
  width:400px;
  height:300px;
  background-color:silver;
  border:1px solid silver;
}
.container ul{
  width:100%;
  height:100%;
  overflow:hidden;
}
.container .item{
  float:left;
  width:25%;
  height:100%;
  background-color:white;
}
.container .item .title{
  line-height:40px;
  border:1px solid silver;
  box-sizing:border-box;
  text-align:center;
  cursor:pointer;
}
.container .item .content{
  width:400%;
  height:100%;
  background-color:yellow;
}
.ml1{
  margin-left:-100%;
}
.ml2{
  margin-left:-200%;
}
.ml3{
  margin-left:-300%;
}
.active{
  position:relative;
  z-index:1
}
.container .item:hover{
  position:relative;
  z-index:1
}
.container .item:hover .title{
  border-bottom:none;
  background-color:yellow;
}
```
#### 3、label与:checked实现
**（1）针对布局一：**
```
<div class="container">
  <div class="tab-content">
    <input type="radio" name="item" class="radio-item" id="item1" checked/>
    <div class="item">内容1</div>
    <input type="radio" name="item" class="radio-item" id="item2" />
    <div class="item">内容2</div>
    <input type="radio" name="item" class="radio-item" id="item3" />
    <div class="item">内容3</div>
    <input type="radio" name="item" class="radio-item" id="item4" />
    <div class="item">内容4</div>
  </div>
  <div class="tab-control">
    <ul>
      <li><label for="item1">内容1</label></li>
      <li><label for="item2">内容2</label></li>
      <li><label for="item3">内容3</label></li>
      <li><label for="item4">内容4</label></li>
    </ul>
  </div>
</div>
```
```
ul,
li {
  margin: 0;
  padding: 0;
  list-style: none;
}

.container {
  width: 400px;
  height: 300px;
  background-color: silver;
}

.tab-content {
  position: relative;
  width: 100%;
  height: 80%;
  overflow: hidden;
}

input {
  margin: 0;
  width: 0;
}

.tab-content .item {
  position: absolute;
  left: 0;
  top: 0;
  width: 100%;
  height: 100%;
}

.tab-control {
  width: 100%;
  height: 20%;
}

.tab-control ul {
  height: 100%;
}

.tab-control li {
  width: 25%;
  height: 100%;
  float: left;
  border: 1px solid silver;
  box-sizing: border-box;
  background-color: white;
}

.tab-control li:hover {
  background-color: #7b7474
}

.tab-control label {
  display: inline-block;
  width: 100%;
  height: 100%;
  line-height: 100%;
  text-align: center;
  text-decoration: none;
  cursor: pointer;
}

.tab-control label::after {
  content: "";
  display: inline-block;
  height: 100%;
  vertical-align: middle;
}
.tab-content .radio-item{
  display:none;
}
.tab-content .radio-item:checked+.item {
  z-index: 1;
  background-color: yellow;
}
```
利用css :checked与+（选择紧接在另一个元素后的元素，而且二者有相同的父元素）选择符。

**（2）针对布局二：**
```
<div class="container">
   <ul>
     <li class="item active">
       <input type="radio" name="item" class="radio-item" id="item1" checked/>
       <label class="title" for="item1">1</label>
       <p class="content">1</p>
     </li>
     <li class="item">
       <input type="radio" name="item" class="radio-item" id="item2" />
       <label class="title" for="item2">2</label>
       <p class="content ml1">2</p>
     </li>
     <li class="item">
       <input type="radio" name="item" class="radio-item" id="item3" />
       <label class="title" for="item3">3</label>
       <p class="content ml2">3</p>
     </li>
     <li class="item">
       <input type="radio" name="item" class="radio-item" id="item4" />
       <label class="title" for="item4">4</label>
       <p class="content ml3">4</p>
     </li>
   </ul>
</div>
```
```
ul,li,p{
  margin:0;
  padding:0;
  list-style:none;
}
.container{
  width:400px;
  height:300px;
  background-color:silver;
  border:1px solid silver;
}
.container ul{
  width:100%;
  height:100%;
  overflow:hidden;
}
.container .item{
  float:left;
  width:25%;
  height:100%;
  background-color:white;
}
.container .item .title{
  display:inline-block;
  width:100%;
  line-height:40px;
  border:1px solid silver;
  box-sizing:border-box;
  text-align:center;
  cursor:pointer;
}
.container .item .content{
  position:relative;
  width:400%;
  height:100%;
  background-color:yellow;
}
.ml1{
  margin-left:-100%;
}
.ml2{
  margin-left:-200%;
}
.ml3{
  margin-left:-300%;
}
.radio-item{
  display:none;
}
.radio-item:checked~.title{
  background-color:yellow;
  border-bottom:none;
}
.radio-item:checked~.content{
  background-color:yellow;
  z-index:1;
}
```