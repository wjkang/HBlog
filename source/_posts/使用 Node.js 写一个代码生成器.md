title: 使用 Node.js 写一个代码生成器
author: 若邪
tags:
  - Node.js
categories:
  - 后端
date: 2019-05-09 00:00:00
---

## 原理

代码生成器的原理就是：`数据 + 模板 => 文件`。

`数据`一般为数据库的表字段结构。

`模板`的语法与使用的模板引擎有关。

使用模板引擎将`数据`和`模板`进行编译，编译后的内容输出到文件中就得到了一份代码文件。

## 功能

因为这个代码生成器是要集成到一个小工具 [lazy-mock](https://github.com/wjkang/lazy-mock) 内，这个工具的主要功能是启动一个 mock server 服务，包含curd功能，并且支持数据的持久化，文件变化的时候自动重启服务以最新的代码提供 api mock 服务。

代码生成器的功能就是根据配置的数据和模板，编译后将内容输出到指定的目录文件中。因为添加了新的文件，mock server 服务会自动重启。

还要支持模板的定制与开发，以及使用 CLI 安装模板。

可以开发前端项目的模板，直接将编译后的内容输出到前端项目的相关目录下，webpack 的热更新功能也会起作用。

## 模板引擎

模板引擎使用的是 [nunjucks](http://mozilla.github.io/nunjucks/getting-started.html)。

[lazy-mock](https://github.com/wjkang/lazy-mock) 使用的构建工具是 gulp，使用 gulp-nodemon 实现 mock-server 服务的自动重启。所以这里使用 gulp-nunjucks-render 配合 gulp 的构建流程。

## 代码生成

编写一个 gulp task ：

```js
const rename = require('gulp-rename')
const nunjucksRender = require('gulp-nunjucks-render')
const codeGenerate = require('./templates/generate')
const ServerFullPath = require('./package.json').ServerFullPath; //mock -server项目的绝对路径
const FrontendFullPath = require('./package.json').FrontendFullPath; //前端项目的绝对路径
const nunjucksRenderConfig = {
  path: 'templates/server',
  envOptions: {
    tags: {
      blockStart: '<%',
      blockEnd: '%>',
      variableStart: '<$',
      variableEnd: '$>',
      commentStart: '<#',
      commentEnd: '#>'
    },
  },
  ext: '.js',
  //以上是 nunjucks 的配置
  ServerFullPath,
  FrontendFullPath
}
gulp.task('code', function () {
  require('events').EventEmitter.defaultMaxListeners = 0
  return codeGenerate(gulp, nunjucksRender, rename, nunjucksRenderConfig)
});
```
> 代码具体结构细节可以打开 [lazy-mock](https://github.com/wjkang/lazy-mock) 进行参照

为了支持模板的开发，以及更灵活的配置，我将代码生成的逻辑全都放在模板目录中。

`templates` 是存放模板以及数据配置的目录。结构如下：

![](https://user-gold-cdn.xitu.io/2019/5/10/16a9e71ad9b2e8b1?w=433&h=660&f=png&s=27198)

只生成 lazy-mock 代码的模板中 ：

`generate.js`的内容如下：

```js
const path = require('path')
const CodeGenerateConfig = require('./config').default;
const Model = CodeGenerateConfig.model;

module.exports = function generate(gulp, nunjucksRender, rename, nunjucksRenderConfig) {
    nunjucksRenderConfig.data = {
        model: CodeGenerateConfig.model,
        config: CodeGenerateConfig.config
    }
    const ServerProjectRootPath = nunjucksRenderConfig.ServerFullPath;
    //server
    const serverTemplatePath = 'templates/server/'
    gulp.src(`${serverTemplatePath}controller.njk`)
        .pipe(nunjucksRender(nunjucksRenderConfig))
        .pipe(rename(Model.name + '.js'))
        .pipe(gulp.dest(ServerProjectRootPath + CodeGenerateConfig.config.ControllerRelativePath));

    gulp.src(`${serverTemplatePath}service.njk`)
        .pipe(nunjucksRender(nunjucksRenderConfig))
        .pipe(rename(Model.name + 'Service.js'))
        .pipe(gulp.dest(ServerProjectRootPath + CodeGenerateConfig.config.ServiceRelativePath));

    gulp.src(`${serverTemplatePath}model.njk`)
        .pipe(nunjucksRender(nunjucksRenderConfig))
        .pipe(rename(Model.name + 'Model.js'))
        .pipe(gulp.dest(ServerProjectRootPath + CodeGenerateConfig.config.ModelRelativePath));

    gulp.src(`${serverTemplatePath}db.njk`)
        .pipe(nunjucksRender(nunjucksRenderConfig))
        .pipe(rename(Model.name + '_db.json'))
        .pipe(gulp.dest(ServerProjectRootPath + CodeGenerateConfig.config.DBRelativePath));

    return gulp.src(`${serverTemplatePath}route.njk`)
        .pipe(nunjucksRender(nunjucksRenderConfig))
        .pipe(rename(Model.name + 'Route.js'))
        .pipe(gulp.dest(ServerProjectRootPath + CodeGenerateConfig.config.RouteRelativePath));
}
```

类似：

```js
gulp.src(`${serverTemplatePath}controller.njk`)
        .pipe(nunjucksRender(nunjucksRenderConfig))
        .pipe(rename(Model.name + '.js'))
        .pipe(gulp.dest(ServerProjectRootPath + CodeGenerateConfig.config.ControllerRelativePath));
```
表示使用 controller.njk 作为模板，nunjucksRenderConfig作为数据（模板内可以获取到 nunjucksRenderConfig 属性 data 上的数据）。编译后进行文件重命名，并保存到指定目录下。

`model.js` 的内容如下：

```js
var shortid = require('shortid')
var Mock = require('mockjs')
var Random = Mock.Random

//必须包含字段id
export default {
    name: "book",
    Name: "Book",
    properties: [
        {
            key: "id",
            title: "id"
        },
        {
            key: "name",
            title: "书名"
        },
        {
            key: "author",
            title: "作者"
        },
        {
            key: "press",
            title: "出版社"
        }
    ],
    buildMockData: function () {//不需要生成设为false
        let data = []
        for (let i = 0; i < 100; i++) {
            data.push({
                id: shortid.generate(),
                name: Random.cword(5, 7),
                author: Random.cname(),
                press: Random.cword(5, 7)
            })
        }
        return data
    }
}
```
模板中使用最多的就是这个数据，也是生成新代码需要配置的地方，比如这里配置的是 book ，生成的就是关于 book 的curd 的 mock 服务。要生成别的，修改后执行生成命令即可。

buildMockData 函数的作用是生成 mock 服务需要的随机数据，在 db.njk 模板中会使用：

```
{
  "<$ model.name $>":<% if model.buildMockData %><$ model.buildMockData()|dump|safe $><% else %>[]<% endif %>
}
```

>这也是 nunjucks 如何在模板中执行函数

`config.js` 的内容如下：
```js
export default {
    //server
    RouteRelativePath: '/src/routes/',
    ControllerRelativePath: '/src/controllers/',
    ServiceRelativePath: '/src/services/',
    ModelRelativePath: '/src/models/',
    DBRelativePath: '/src/db/'
}
```
配置相应的模板编译后保存的位置。

`config/index.js` 的内容如下：

```js
import model from './model';
import config from './config';
export default {
    model,
    config
}
```
针对 lazy-mock 的代码生成的功能就已经完成了，要实现模板的定制直接修改模板文件即可，比如要修改 mock server 服务 api 的接口定义，直接修改 route.njk 文件：

```js
import KoaRouter from 'koa-router'
import controllers from '../controllers/index.js'
import PermissionCheck from '../middleware/PermissionCheck'

const router = new KoaRouter()
router
    .get('/<$ model.name $>/paged', controllers.<$model.name $>.get<$ model.Name $>PagedList)
    .get('/<$ model.name $>/:id', controllers.<$ model.name $>.get<$ model.Name $>)
    .del('/<$ model.name $>/del', controllers.<$ model.name $>.del<$ model.Name $>)
    .del('/<$ model.name $>/batchdel', controllers.<$ model.name $>.del<$ model.Name $>s)
    .post('/<$ model.name $>/save', controllers.<$ model.name $>.save<$ model.Name $>)

module.exports = router
```

## 模板开发与安装

不同的项目，代码结构是不一样的，每次直接修改模板文件会很麻烦。

需要提供这样的功能：针对不同的项目开发一套独立的模板，支持模板的安装。

代码生成的相关逻辑都在模板目录的文件中，模板开发没有什么规则限制，只要保证目录名为 `templates`，`generate.js`中导出`generate`函数即可。

模板的安装原理就是将模板目录中的文件全部覆盖掉即可。不过具体的安装分为本地安装与在线安装。

之前已经说了，这个代码生成器是集成在 [lazy-mock](https://github.com/wjkang/lazy-mock) 中的，我的做法是在初始化一个新 [lazy-mock](https://github.com/wjkang/lazy-mock) 项目的时候，指定使用相应的模板进行初始化，也就是安装相应的模板。

使用 Node.js 写了一个 CLI 工具 [lazy-mock-cli](https://github.com/wjkang/lazy-mock-cli)，已发到 npm ，其功能包含下载指定的远程模板来初始化新的 lazy-mock 项目。代码参考（ copy ）了 [vue-cli2](https://github.com/vuejs/vue-cli/tree/v2)。代码不难，说下某些关键点。

安装 CLI 工具：

```bash
npm install lazy-mock -g
```
使用模板初始化项目：
```bash
lazy-mock init d2-admin-pm my-project
```

> [d2-admin-pm](https://github.com/lazy-mock-templates/d2-admin-pm) 是我为一个[前端项目](https://github.com/wjkang/d2-admin-pm)已经写好的一个模板。

`init` 命令调用的是 [lazy-mock-init.js](https://github.com/wjkang/lazy-mock-cli/blob/master/bin/lazy-mock-init.js) 中的逻辑：
```js
#!/usr/bin/env node
const download = require('download-git-repo')
const program = require('commander')
const ora = require('ora')
const exists = require('fs').existsSync
const rm = require('rimraf').sync
const path = require('path')
const chalk = require('chalk')
const inquirer = require('inquirer')
const home = require('user-home')
const fse = require('fs-extra')
const tildify = require('tildify')
const cliSpinners = require('cli-spinners');
const logger = require('../lib/logger')
const localPath = require('../lib/local-path')

const isLocalPath = localPath.isLocalPath
const getTemplatePath = localPath.getTemplatePath

program.usage('<template-name> [project-name]')
    .option('-c, --clone', 'use git clone')
    .option('--offline', 'use cached template')

program.on('--help', () => {
    console.log('  Examples:')
    console.log()
    console.log(chalk.gray('    # create a new project with an official template'))
    console.log('    $ lazy-mock init d2-admin-pm my-project')
    console.log()
    console.log(chalk.gray('    # create a new project straight from a github template'))
    console.log('    $ vue init username/repo my-project')
    console.log()
})

function help() {
    program.parse(process.argv)
    if (program.args.length < 1) return program.help()
}
help()
//模板
let template = program.args[0]
//判断是否使用官方模板
const hasSlash = template.indexOf('/') > -1
//项目名称
const rawName = program.args[1]
//在当前文件下创建
const inPlace = !rawName || rawName === '.'
//项目名称
const name = inPlace ? path.relative('../', process.cwd()) : rawName
//创建项目完整目标位置
const to = path.resolve(rawName || '.')
const clone = program.clone || false

//缓存位置
const serverTmp = path.join(home, '.lazy-mock', 'sever')
const tmp = path.join(home, '.lazy-mock', 'templates', template.replace(/[\/:]/g, '-'))
if (program.offline) {
    console.log(`> Use cached template at ${chalk.yellow(tildify(tmp))}`)
    template = tmp
}

//判断是否当前目录下初始化或者覆盖已有目录
if (inPlace || exists(to)) {
    inquirer.prompt([{
        type: 'confirm',
        message: inPlace
            ? 'Generate project in current directory?'
            : 'Target directory exists. Continue?',
        name: 'ok'
    }]).then(answers => {
        if (answers.ok) {
            run()
        }
    }).catch(logger.fatal)
} else {
    run()
}

function run() {
    //使用本地缓存
    if (isLocalPath(template)) {
        const templatePath = getTemplatePath(template)
        if (exists(templatePath)) {
            generate(name, templatePath, to, err => {
                if (err) logger.fatal(err)
                console.log()
                logger.success('Generated "%s"', name)
            })
        } else {
            logger.fatal('Local template "%s" not found.', template)
        }
    } else {
        if (!hasSlash) {
            //使用官方模板
            const officialTemplate = 'lazy-mock-templates/' + template
            downloadAndGenerate(officialTemplate)
        } else {
            downloadAndGenerate(template)
        }
    }
}

function downloadAndGenerate(template) {
    downloadServer(() => {
        downloadTemplate(template)
    })
}

function downloadServer(done) {
    const spinner = ora('downloading server')
    spinner.spinner = cliSpinners.bouncingBall
    spinner.start()
    if (exists(serverTmp)) rm(serverTmp)
    download('wjkang/lazy-mock', serverTmp, { clone }, err => {
        spinner.stop()
        if (err) logger.fatal('Failed to download server ' + template + ': ' + err.message.trim())
        done()
    })
}

function downloadTemplate(template) {
    const spinner = ora('downloading template')
    spinner.spinner = cliSpinners.bouncingBall
    spinner.start()
    if (exists(tmp)) rm(tmp)
    download(template, tmp, { clone }, err => {
        spinner.stop()
        if (err) logger.fatal('Failed to download template ' + template + ': ' + err.message.trim())
        generate(name, tmp, to, err => {
            if (err) logger.fatal(err)
            console.log()
            logger.success('Generated "%s"', name)
        })
    })
}

function generate(name, src, dest, done) {
    try {
        fse.removeSync(path.join(serverTmp, 'templates'))
        const packageObj = fse.readJsonSync(path.join(serverTmp, 'package.json'))
        packageObj.name = name
        packageObj.author = ""
        packageObj.description = ""
        packageObj.ServerFullPath = path.join(dest)
        packageObj.FrontendFullPath = path.join(dest, "front-page")
        fse.writeJsonSync(path.join(serverTmp, 'package.json'), packageObj, { spaces: 2 })
        fse.copySync(serverTmp, dest)
        fse.copySync(path.join(src, 'templates'), path.join(dest, 'templates'))
    } catch (err) {
        done(err)
        return
    }
    done()
}
```

判断了是使用本地缓存的模板还是拉取最新的模板，拉取线上模板时是从官方仓库拉取还是从别的仓库拉取。

## 一些小问题

目前代码生成的相关数据并不是来源于数据库，而是在 `model.js` 中简单配置的，原因是我认为一个 mock server 不需要数据库，lazy-mock 确实如此。

但是如果写一个正儿八经的代码生成器，那肯定是需要根据已经设计好的数据库表来生成代码的。那么就需要连接数据库，读取数据表的字段信息，比如字段名称，字段类型，字段描述等。而不同关系型数据库，读取表字段信息的 sql 是不一样的，所以还要写一堆balabala的判断。可以使用现成的工具 [sequelize-auto](https://github.com/sequelize/sequelize-auto) , 把它读取的 model 数据转成我们需要的格式即可。

生成前端项目代码的时候，会遇到这种情况：

某个目录结构是这样的：


![](https://user-gold-cdn.xitu.io/2019/5/10/16a9ef45e3963c64?w=660&h=380&f=png&s=14236)

`index.js` 的内容：

```js
import layoutHeaderAside from '@/layout/header-aside'
export default {
    "layoutHeaderAside": layoutHeaderAside,
    "menu": () => import(/* webpackChunkName: "menu" */'@/pages/sys/menu'),
    "route": () => import(/* webpackChunkName: "route" */'@/pages/sys/route'),
    "role": () => import(/* webpackChunkName: "role" */'@/pages/sys/role'),
    "user": () => import(/* webpackChunkName: "user" */'@/pages/sys/user'),
    "interface": () => import(/* webpackChunkName: "interface" */'@/pages/sys/interface')
}
```
如果添加一个 book 就需要在这里加上`"book": () => import(/* webpackChunkName: "book" */'@/pages/sys/book')`

这一行内容也是可以通过配置模板来生成的，比如模板内容为：

```
"<$ model.name $>": () => import(/* webpackChunkName: "<$ model.name $>" */'@/pages<$ model.module $><$ model.name $>')
```
但是生成的内容怎么加到`index.js`中呢？

第一种方法：复制粘贴

第二种方法：

这部分的模板为 [routerMapComponent.njk](https://github.com/lazy-mock-templates/d2-admin-pm/blob/master/templates/front-end/routerMapComponent.njk) ：

```
export default {
    "<$ model.name $>": () => import(/* webpackChunkName: "<$ model.name $>" */'@/pages<$ model.module $><$ model.name $>')
}
```

编译后文件保存到 routerMapComponents 目录下，比如 book.js

修改 index.js :

```js
const files = require.context('./', true, /\.js$/);
import layoutHeaderAside from '@/layout/header-aside'

let componentMaps = {
    "layoutHeaderAside": layoutHeaderAside,
    "menu": () => import(/* webpackChunkName: "menu" */'@/pages/sys/menu'),
    "route": () => import(/* webpackChunkName: "route" */'@/pages/sys/route'),
    "role": () => import(/* webpackChunkName: "role" */'@/pages/sys/role'),
    "user": () => import(/* webpackChunkName: "user" */'@/pages/sys/user'),
    "interface": () => import(/* webpackChunkName: "interface" */'@/pages/sys/interface'),
}
files.keys().forEach((key) => {
    if (key === './index.js') return
    Object.assign(componentMaps, files(key).default)
})
export default componentMaps
```
>使用了 require.context

我目前也是使用了这种方法

第三种方法：

开发模板的时候，做特殊处理，读取原有 index.js 的内容，按行进行分割，在数组的最后一个元素之前插入新生成的内容，注意逗号的处理，将新数组内容重新写入 index.js 中，注意换行。




