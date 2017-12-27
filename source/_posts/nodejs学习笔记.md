
title: nodejs学习笔记
date: 2016-12-05 17:08:17
tags: [youdaonote]
---

v8的出现，还有commonjs的规范化使得nodejs提供后台服务成为可能。

两种启动方式：
- node app.js   不自动更新
- supervisor app.js 自动更新，修改后不用重启

---
特点
---
- 默认全部都是异步操作
```js
var fs = require('fs');
fs.readFile('test.file', 'utf-8', function(err, data){
    if (err) {
        console.log(err);
    } else {
        console.log(data);
    }
});
console.log('end'); // 这一行会先被打印出来
```
- 全部基于事件处理机制，nodejs的所有机制都是围绕事件回调机制完成的。事件的回调函数在执行的过程中，可能会发出IO请求或者emit事件，执行完毕后再返回事件循环
```js
// 声明事件对象
var EventEmitter=require('events').EventEmiiter;
var event = new EventEmiiter();

//注册事件
event.on('some_event', function(){
    console.log("this is a custom event");
});


// 3秒钟后触发事件
setTimeout(function(){
    event.emit('some_event');
}, 3000);
```
--- 
module和package
---
nodejs的module可以是js、json或者c++扩展。
```js
var http = require('http'); // c++的module
```
exports是模块对外开放的接口
```js
var name;
exports.setName = function(n){
    name = n;
}

exports.say = function(){
    console.log('name is : ' + name);
}
```
require是外部引用模块的接口, 只会引入一次
```js
var modu1=require('./modu');
modu1.setName('will');
var modu2=require('./modu');
modu2.setName('william');
modu2.say(); // 会输出william
```
如果我们要每次都弄一个新的对象呢??
```js
function hello() {
    var name;
    this.setName = function(n){
        name = n;
    }
    
    this.say = function(){
        console.log('name is : ' + name);
    }
}
module.exports = hello;
```
对应的引入为：
```js
var hello=require('./modu');
var modu1 = new hello();
modu1.setName('will');
var modu2 = new hello();
modu2.setName('william');
modu1.say(); // 会输出will
modu2.say(); // 会输出william
```

### pacakge
commonJS规范后，可以发布在npm上
 - package.json在根目录下，没有的话就去找index.js
 - 二进制文件在bin目录下
 - js代码在lib目录下
 - 测试在test目录
 - ....

```json
// package.json
{
    "main" : "./lib/index.js", // 入口
    "name" : "名",
    "repositories": "仓库托管地址数据，每个元素要包含type、url、path",
    "dependencies" : "依赖"
}
```

---
包管理器和代码调试
---

#### 包管理器 

|命令|介绍|
|---|---|
|npm init| 初始化一个package目录，会自动生成规范目录与package.json |
|npm install xxx| 本地模式安装xxx，将package安装到node_modules目录下|
|npm list| 查看已经安装的包|
|npm help <json>|查看json的使用|
|npm publish | 进入到package下执行，发布package到公共库（最好是通过npm init生成的目录规范） |
|npm unpublish |取消发布|

***全局安装的包，不能直接require，不会搜索/usr/local/lib/node_modules。只有需要命令行使用的时候才安装全局模式***

#### 代码调试

1. node --debug-brk=5858 app.js, 会等待debug连上来
2. eclipse插件V8 vm里弄一个debug就行了



---
全局对象和全局变量
---
global是所有全局变量的宿主。

#### console
log\error\trace
#### process
描述当前进程状态，提供了与os交互的接口。
|变量|描述|
|---|---|
|process.argv|命令行参数，第一个是node，第二是脚本名称，第三个才是参数|
|process.stdout|标准输出|
|process.stdin|标准输入，需要先process.stdin.resume()|
|process.nextTick(callback)|为事件循环设置一个任务|
nodejs适合io密集型的应用，而不是计算密集型的应用。

```js   
function doSth(args, callback){
    sthComplext(args);
    callback();
};

doSth('123', function onEnd(){
    compute();
});
```
有了process.nextTick后可以写成：
```js
function doSth(args, callback){
    sthComplext(args);
    process.nextTick(callback); // 把耗时的两个程序拆成两个事件，提高并行度
}
```


---
util和EventEmitter
---
##### utilities
|api|描述|
|---|---|
|inheris(constructor, superConstructor)|实现对象间原型继承|
|inspect()|将任意对象以字符串的形式打印出来|

##### events
nodejs本身架构就是事件驱动的。
EventEmitter的核心就是事件发射与事件监听功能的封装。EventEmitter的每个事件由一个事件或若干个参数组成。对于某个事件，EventEmitter支持若干个事件监听器。事件发生时，事件监听器被依次调用，事件参数作为回调函数的参数传递。
|api|描述|
|---|---|
|EventEmitter.on(event, listener)|为指定事件注册监听器|
|EventEmitter.emit()|发射event事件|
|EventEmitter.once(event, listener)|监听器只出发一次|
|EventEmitter.removeListener(event, listener)|移除监听器|
|EventEmitter.removeAllListener(event)|移除所有监听器|

特殊事件error，出错的时候，EventEmitter没有注册相应的监听器，就当做异常，退出程序，打印调用栈。

一般继承EventEmitter来使用它。 
