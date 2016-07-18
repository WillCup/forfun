title: ember散记
date: 2016-06-26 09:44:57
tags: [ember]
---

主要是弄ambari的时候，ambari view使用了emberjs框架，自己按照官网教程走了一遍，此文章只是记录一些散记，不成系统。

命令
---
命令|解释|扩展
---|---|---
ember g route xxx/contact|生成route handler| url里访问的路径xxx/contact会被定位到routes/xxx/contact.js下。生成template文件，route文件，对应的单元测试文件
ember g component list | 生成一个component | 位于templates/components文件夹下，可以被其他template以模块的形式引用。对应的js文件在根目录的components下
ember install ember-cli-tutorial-style | 安装插件到当前项目 | 示例中的这个插件，会自动copy一些css文件到指定目录
ember g model rental | 生成一个ember data model | 在models文件夹下生成对应的model文件还有单元测试文件 
ember g helper rental-property-type | 生成一个helper | 在helpers文件夹下生成对应的js文件，还有单元测试文件
ember g controller index | 生成一个controller| 在controllers文件夹下生成对应js文件，单元测试文件

路由
---

route handler中添加model方法，可以在template里直接使用。

```javascript
export default Ember.Route.extend({
  model() {
    return rentals;            
  }
});
```

在template中的使用：

```html
[[#each model as |rental|]]    
  <article class="listing">    
    <h3>[[rental.title]]</h3>  
    <div class="detail owner"> 
      <span>Owner:</span> [[rental.owner]]
    </div>
    <div class="detail type">  
      <span>Type:</span> [[rental.type]]
    </div>
    <div class="detail location">   
      <span>Location:</span> [[rental.city]]
    </div>
    <div class="detail bedrooms">   
      <span>Number of bedrooms:</span> [[rental.bedrooms]]
    </div>
  </article>
[[/each]]
~         
```

模板
---
随着项目的创建，会在templates下自动生成一个application.js文件：
```
<h2>title or slogin</h2>

[[outlet]]
```
其中h2标题部分会在所有的模板中出现，[[outlet]]作为url中指定route handler的占位符。

插件
---
ember有自己的插件机制，可以自由发布与引用,个人感觉有点儿类似python module的感觉。

https://emberobserver.com/

Ember data
---
使用ember g model rental生成模板文件models/rental.js
```js
import Model from 'ember-data/model';
import attr from 'ember-data/attr';

export default Model.extend({
  title: attr(),
  owner: attr(),
  city: attr(),
  type: attr(),
  image: attr(),
  bedrooms: attr()
});
```
上面就定义好了这个model的各个属性。下面是使用model了，在某个route handler中定义model函数，下面这个示例会使用GET方法调用/rentals
```js
import Ember from 'ember';

export default Ember.Route.extend({
  model() {
    return this.store.findAll('rental');
  }
});
```
详情参考：https://guides.emberjs.com/v2.6.0/models/


模块
---
就是component，可以被其他的template直接引用。它包含两部分：template和js。
template负责布局，js负责定义一些属性和action。

布局文件:templates/components/rental-listing.hbs
```html
<article class="listing">
  <a [[action 'toggleImageSize']] class="image [[if isWide "wide"]]">
    <img src="[[rental.image]]" alt="">
    <small>View Larger</small>
  </a>
  <h3>[[rental.title]]</h3>
  <div class="detail owner">
    <span>Owner:</span> [[rental.owner]]
  </div>
  <div class="detail type">
    <span>Type:</span> [[rental.type]]
  </div>
  <div class="detail location">
    <span>Location:</span> [[rental.city]]
  </div>
  <div class="detail bedrooms">
    <span>Number of bedrooms:</span> [[rental.bedrooms]]
  </div>
</article>

```
交互文件：components/rental-listing.js
```js
import Ember from 'ember';

export default Ember.Component.extend({
  isWide: false,
  actions: {
    toggleImageSize() {
      this.toggleProperty('isWide');
    }
  }
});
```

可以看到isWide和actions属性与模板文件中的[[action 'toggleImageSize']]以及isWide变量对应。

Handlebars Helper
---
有的时候会在render到template之前对一些属性进行公共的处理啥的，就用到这个handlebar，为属性起装饰作用。
helpers/xxx.js文件：
```javascript
import Ember from 'ember';

const communityPropertyTypes = [
  'Condo',
  'Townhouse',
  'Apartment'
];

export function rentalPropertyType([type]/*, hash*/) {
  if (communityPropertyTypes.contains(type)) {
    return 'Community';
  }

  return 'Standalone';
}

export default Ember.Helper.helper(rentalPropertyType);
```
在template文件中使用上面定义的rentalPropertyType方法。
```html
 <span>Type:</span> [[rental-property-type rental.type]] - [[rental.type]]
```

这样在render模板的时候，就会把rental.type传入rentalPropertyType函数，处理后，返回相应的处理值。
