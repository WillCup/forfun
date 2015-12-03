title: node杂记
date: 2015-10-15 15:48:35
tags:
---

因为新的项目这边前端是使用的nodejs，精通自己，熟知周边是我一直依赖的坚持。所以现在要对nodejs有个hello world的了解，便于以后自己调试自己的接口以及其他功能代码。

nodejs的依赖管理工具是npm, 有些类似Python的pip + java的maven，既负责安装全局性的依赖，也能处理当前项目的依赖。



- npm install xxx 是安装依赖到当前目录的node_modules文件夹下，只有当前项目能够使用这些依赖；
- npm install xxx -g 安装依赖到npm根目录的node_modules文件夹下，本机所有nodejs项目都可以使用

npm主要针对package.json进行，其实npm有啥功能只要看一下package.json有啥配置项就可以了。但是为了完整性，还是列一下:

|*命令*|*描述* |
|---|:---|
|npm install|安装package.json里指定的所有依赖|
|npm install xxx|安装xxx,其实这个命令感觉跟当前项目根本没啥关系了|
|npm install xxx  --save|安装xxx,同时将xxx的最新版本加入package.json的dependency里|
|npm set init-author-name 'Your name'|等于为npm init设置了默认值，以后执行npm init的时候，package.<br>json的作者姓名、邮件、主页、许可证字段就会自动写入预设的值。这些信息会存放在用户主目录的~/.npmrc文件，使得用户不用每个项目都输入。如果某个项目有不同的设置，可以针对该项目运行npm config。|
|npm info xxx |查看模块xxx的具体信息|
|npm search xxx |查找xxx模块是否存在于npm仓库|
|npm list |树形结构列出当前项目安装的所有module|
|npm info xxx |查看模块xxx的具体信息|


注： nodejs自身还有版本管理工具，就是nvm了，可以在一个机器上安装多个版本的nodejs.

>  git clone https://github.com/creationix/nvm.git ~/.nvm
>  source ~/.nvm/nvm.sh# 

 安装最新版本
> nvm install node

 安装指定版本
> nvm install 0.12.1

# 使用已安装的最新版本
> nvm use node

# 使用指定版本的node
> nvm use 0.12

# 查看本地安装的所有版本
>  nvm ls

# 查看服务器上所有可供安装的版本。
> nvm ls-remote

# 退出已经激活的nvm，使用deactivate命令。
> nvm deactivate




nodejs是一种事件驱动的思想：nodejs的event流有些像java web这边的servlet chain，不过他们的servlet叫做middleware，而且很多层次都可以有middleware。

### app
```
// a middleware sub-stack which handles GET requests to /user/:id
app.get('/user/:id', function (req, res, next) {
  // if user id is 0, skip to the next route
  if (req.params.id == 0) next('route');
  // else pass the control to the next middleware in this stack
  else next(); //
}, function (req, res, next) {
  // render a regular page
  res.render('regular');
});

// handler for /user/:id which renders a special page
app.get('/user/:id', function (req, res, next) {
  res.render('special');
});
```
### router
```
var app = express();
var router = express.Router();

// a middleware with no mount path, gets executed for every request to the router
router.use(function (req, res, next) {
  console.log('Time:', Date.now());
  next();
});

// a middleware sub-stack shows request info for any type of HTTP request to /user/:id
router.use('/user/:id', function(req, res, next) {
  console.log('Request URL:', req.originalUrl);
  next();
}, function (req, res, next) {
  console.log('Request Type:', req.method);
  next();
});

// a middleware sub-stack which handles GET requests to /user/:id
router.get('/user/:id', function (req, res, next) {
  // if user id is 0, skip to the next router
  if (req.params.id == 0) next('route');
  // else pass the control to the next middleware in this stack
  else next(); //
}, function (req, res, next) {
  // render a regular page
  res.render('regular');
});

// handler for /user/:id which renders a special page
router.get('/user/:id', function (req, res, next) {
  console.log(req.params.id);
  res.render('special');
});

// mount the router on the app
app.use('/', router);
```
### error handler
四个参数，多一个err。如果完事儿不调用next的话，就需要指定状态码，不然就会向普通的middleware一样报错，并且执行失败。
```
app.use(function(err, req, res, next) {
  console.error(err.stack);
  res.status(500).send('Something broke!');
});
```
### built-in
expresss.static(root, [options]) 一般指定应用的静态资源根目录
```
var options = {
  dotfiles: 'ignore',
  etag: false,
  extensions: ['htm', 'html'],
  index: false,
  maxAge: '1d',
  redirect: false,
  setHeaders: function (res, path, stat) {
    res.set('x-timestamp', Date.now());
  }
}

app.use(express.static('public', options));
```
也可以指定多个目录
```
app.use(express.static('public'));
app.use(express.static('uploads'));
app.use(express.static('files'));
```
### third-party
```
var express = require('express');
var app = express();
var cookieParser = require('cookie-parser');

// load the cookie parsing middleware
app.use(cookieParser());
```

node启动express项目：node app.js，如果指定参数-p 8080,就会启动port为8080的server。
process.env是指引用系统的环境变量

jade是一种node的模板引擎，将一些关键字映射为html里的元素，使简洁，而且生成的HTML是最小的、确保正确的。

- 使用的时候可以使好多公共的东西模块化，然后include进来
- 模板之间可以extends，一般extends layout, 然后给里面的一些变量赋值,一般都是处理block
详见官网吧。

bootstrap + jade，you know...

UI: bootstrap
模板引擎：jade
css引擎：stylus
前端：jquery + socket.io
后端：express