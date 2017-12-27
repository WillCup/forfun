
title: ember异常记录
date: 2016-12-05 17:09:02
tags: [youdaonote]
---

#### ember-mermaid
要画流程图，对比了一下，决定使用ember-mermaid，但是github里说的安装方式运行的话，会报错：
```bash
root@will-vm:/usr/local/metamap/metamap_js# ember install ember-mermaid
DEPRECATION: Overriding init without calling this._super is deprecated. Please call this._super(), addon: `ember-mermaid`
    at Function.Addon.lookup (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addon.js:896:27)
The addon `ember-cli-htmlbars` requires an Ember CLI version of 0.1.2 or above, but you are running null.
Error: The addon `ember-cli-htmlbars` requires an Ember CLI version of 0.1.2 or above, but you are running null.
    at Function.deprecatedAssertAbove [as assertAbove] (/usr/local/metamap/metamap_js/node_modules/ember-cli-htmlbars/node_modules/ember-cli-version-checker/index.js:143:18)
    at CoreObject.module.exports.init (/usr/local/metamap/metamap_js/node_modules/ember-cli-htmlbars/ember-addon-main.js:13:13)
    at CoreObject.superWrapper [as init] (/usr/local/metamap/metamap_js/node_modules/ember-cli/node_modules/core-object/lib/assign-properties.js:32:18)
    at CoreObject.Class (/usr/local/metamap/metamap_js/node_modules/ember-cli/node_modules/core-object/core-object.js:33:38)
    at /usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addons-factory.js:48:21
    at visit (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/utilities/DAG.js:23:3)
    at DAG.topsort (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/utilities/DAG.js:82:7)
    at CoreObject.extend.initializeAddons (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addons-factory.js:44:11)
    at CoreObject.extend.initializeAddons (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addon.js:226:38)
    at setupRegistryForEachAddon (/usr/local/metamap/metamap_js/node_modules/ember-cli/node_modules/ember-cli-preprocess-registry/preprocessors.js:18:10)
```

搜了一下没找到相关资料，索性直接暴力解决,修改报错文件里检查版本的部分/usr/local/metamap/metamap_js/node_modules/ember-cli-htmlbars/node_modules/ember-cli-version-checker/index.js:143:18。看到上面是验证版本的地方出了问题
```javascript
  if (!dependencyChecker.satisfies(comparison)) {
    var error  = new Error(message);

    error.suppressStacktrace = true;
    throw error;
  }

```
注释掉上面这些代码后成功安装。
```
root@will-vm:/usr/local/metamap/metamap_js# ember install ember-mermaid
DEPRECATION: Overriding init without calling this._super is deprecated. Please call this._super(), addon: `ember-mermaid`
    at Function.Addon.lookup (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addon.js:896:27)
DEPRECATION: Node v0.10.45 is no longer supported by Ember CLI. Please update to a more recent version of Node
Could not start watchman; falling back to NodeWatcher for file system events.
Visit http://ember-cli.com/user-guide/#watchman for more info.
Installed packages for tooling via npm.
DEPRECATION: Overriding init without calling this._super is deprecated. Please call this._super(), addon: `ember-mermaid`
    at Function.Addon.lookup (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addon.js:896:27)
installing ember-mermaid
  install bower package mermaid
  not-cached https://github.com/knsv/mermaid.git#^0.5.8
  progress 接收对象中:  26% (58/222)
  progress 接收对象中:  67% (149/222), 1.98 MiB | 166.00 KiB/s
  progress 接收对象中:  68% (151/222), 1.98 MiB | 166.00 KiB/s
  resolved https://github.com/knsv/mermaid.git#0.5.8
  conflict Unable to find suitable version for mermaid
    1) mermaid ^0.5.8
    2) mermaid ^6.0.0
? Answer 2
Installed browser packages via Bower.
Installed addon package.

```
既然成功安装了，为防有其他差错，就把这个检查版本的文件恢复回去了。
然后启动ember server又出现了这个问题
```
root@will-vm:/usr/local/metamap/metamap_js# ember serve
DEPRECATION: Overriding init without calling this._super is deprecated. Please call this._super(), addon: `ember-mermaid`
    at Function.Addon.lookup (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addon.js:896:27)
The addon `ember-cli-htmlbars` requires an Ember CLI version of 0.1.2 or above, but you are running null.
Error: The addon `ember-cli-htmlbars` requires an Ember CLI version of 0.1.2 or above, but you are running null.
    at Function.deprecatedAssertAbove [as assertAbove] (/usr/local/metamap/metamap_js/node_modules/ember-cli-htmlbars/node_modules/ember-cli-version-checker/index.js:143:18)
    at CoreObject.module.exports.init (/usr/local/metamap/metamap_js/node_modules/ember-cli-htmlbars/ember-addon-main.js:13:13)
    at CoreObject.superWrapper [as init] (/usr/local/metamap/metamap_js/node_modules/ember-cli/node_modules/core-object/lib/assign-properties.js:32:18)
    at CoreObject.Class (/usr/local/metamap/metamap_js/node_modules/ember-cli/node_modules/core-object/core-object.js:33:38)
    at /usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addons-factory.js:48:21
    at visit (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/utilities/DAG.js:23:3)
    at DAG.topsort (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/utilities/DAG.js:82:7)
    at CoreObject.extend.initializeAddons (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addons-factory.js:44:11)
    at CoreObject.extend.initializeAddons (/usr/local/metamap/metamap_js/node_modules/ember-cli/lib/models/addon.js:226:38)
    at setupRegistryForEachAddon (/usr/local/metamap/metamap_js/node_modules/ember-cli/node_modules/ember-cli-preprocess-registry/preprocessors.js:18:10)

```

仔细看了一下官方github上给出的解释，说是要把ember-font-awesome升级到2.1.1就可以了。查了一下，压根就没有安装这个东西
```
root@will-vm:/usr/local/metamap/metamap_js# find . -name 'ember-font-awesome'
root@will-vm:/usr/local/metamap/metamap_js# 

```
看一下ember-mermaid的官网吧，到ember addon仓库搜索了一把https://emberobserver.com/addons/ember-mermaid，发现人家的ember-cli的版本是2.4.1，最近支持的是2.5.0.群里很多人都说2.6不稳定，所以一直没有升级。。
索性我就降级吧。
```
npm rm -g ember-cli
npm install -g ember-cli@2.5
ember -v
ember new testPro
```
上面使用ember 2.5版本生成了一个临时项目，为了把依赖的版本copy到现有的2.6的项目里去，覆盖掉2.6的那些版本依赖。主要针对bower.json和package.json两个文件里与ember-xxx相关的版本配置。

之后就成功启动了，也能够使用mermaid了，再删掉临时项目就可以了。

`参考` ：
 - https://github.com/crodriguez1a/ember-mermaid
 - https://knsv.github.io/mermaid/#usage
 - http://www.jqueryscript.net/chart-graph/Simple-SVG-Flow-Chart-Plugin-with-jQuery-flowSVG.html
 - https://jsplumbtoolkit.com/community/demo/flowchart/index.html
 - https://www.erp5.com/javascript-10.Flow.Chart
 - https://github.com/ember-cli/ember-cli/issues/5973
