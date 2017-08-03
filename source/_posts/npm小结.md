title: npm小结
author: 若邪
tags:
 - npm
categories:
 - 前端
date: 2016-11-22
---
### 修改公共仓库地址
windows下，修改C:\Users\Administrator下的文件.npmrc，将registry改为淘宝镜像地址，registry=https://registry.npm.taobao.org/。
### 初始化
项目根目录下执行命令行``npm init``，各种参数根据自己需求填写，生成package.json文件。
### 发布自己的npm包
在 npm 官网 [https://www.npmjs.org](https://www.npmjs.org/) 申请一个账号。

``npm init``后，添加要发布的包的相关内容（什么都不加也是可以发布的）。

命令行执行``npm adduser  --registry http://registry.npmjs.org``，输入npm官网申请的账户名，密码和邮箱（因为公共npm仓库已经改为淘宝镜像地址，所以``--registry http://registry.npmjs.org``不能省略）。

执行``npm publish --registry http://registry.npmjs.org``将包发布到npm上。登录到npm官网，点击右上角头像，选中Profile就可以看到发布的包。
### 更新已经发布的包
修改相关的内容后，修改包目录下的package.json文件，修改version版本号，比如之前是1.0.0，修改为2.0.0。然后执行``npm publish --registry http://registry.npmjs.org``就可以将包更新到npm上。
npm install
全局安装``npm install 包名称 -g``

本地安装,并更新package.json的dependencies``npm install 包名称 --save``。

本地安装,并更新package.json的devDependencies``npm install 包名称 --save-dev``。
pm update失败
比如说之前我们发布到npm上的包名称npm-test,版本为1.0.0，未更新之前``npm install npm-test --save-dev``安装的就是1.0.0版本的。新发布了一个可用的版本2.0.0后，想更新到最新的版本，运行``npm update npm-test``,不起作用。原因是因为npm规定了[https://docs.npmjs.com/misc/semver#prerelease-identifiers](https://docs.npmjs.com/misc/semver#prerelease-identifiers)，看不太懂，只知道是这个原因☹
### dependencies和devDependencies的区别
当我们适用别人的包的时候，经常会疑惑，是该``npm install xxx --save``还是``npm install xxx --save-dev``。前者是安装包并且更新package.json的dependencies，后者是安装包并更新package.json的devDependencies，如果只是使用别人的包，而不将包发布到npm上，两种方式都可以正常使用。但是当我们要把我们的包发布到npm上时，一定要区分好dependencies和devDependencies的区别。
比方说我们之前发布的npm-test包，包中需要使用了vue，那是要执行``npm install vue --save``还是``npm install vue --save-dev``呢？当别人使用我们这个包的时候，前提vue这个包已经安装了，但是用户不会去看我们的包依赖哪些包，然后一一得手动去安装这些依赖包。所以当用户 ``npm insall npm-test --save``要使用我们这个包的时候，应该也要自动安装vue这个包。而当我们的包中安装vue的时候，使用``npm install vue --save``就可以了，用户使用我们的包的时候，也会安装vue这个包。如果我们包的相关的功能代码是用es6写的，并且安装了babel,webpack等相关的包，这些包的安装应该使用``npm install vue --save-dev``，当用户使用我们的包的时候，不会安装这些包。