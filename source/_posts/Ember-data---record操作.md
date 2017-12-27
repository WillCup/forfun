
title: Ember-data---record操作
date: 2016-12-05 17:09:52
tags: [youdaonote]
---

查询record
---

### 抽取单个record

使用 [`store.findRecord()`](http://emberjs.com/api/data/classes/DS.Store.html#method_findRecord)方法，按照id和type进行查询. 返回一个包含对应record的promise:

```javascript
var post = this.store.findRecord('post', 1); // => GET /posts/1
```

使用 [`store.peekRecord()`](http://emberjs.com/api/data/classes/DS.Store.html#method_peekRecord).跟上面的一样，但是只查询store中当前已经存在的record，不会向后端server请求:

```javascript
var post = this.store.peekRecord('post', 1); // => no network request
```

### 抽取多个Records

使用 [`store.findAll()`](http://emberjs.com/api/data/classes/DS.Store.html#method_findAll)查询某个类型的所有record:

```javascript
var posts = this.store.findAll('post'); // => GET /posts
```

参上： [`store.peekAll()`](http://emberjs.com/api/data/classes/DS.Store.html#method_peekAll):

```javascript
var posts = this.store.peekAll('post'); // => no network request
```

`store.findAll()` 返回的`PromiseArray` 包含
`RecordArray` 。 `store.peekAll`直接返回`RecordArray`.

注意`RecordArray`并不是JavaScript array，而是一个实现了[`Ember.Enumerable`][1]的对象，所以如果要按照index访问数据，不能使用 `[]`，而应该使用 `objectAt(index)` .

[1]: http://emberjs.com/api/classes/Ember.Enumerable.html

### 查询多个Records

可以查一些满足某些范式的record。
[`store.query()`](http://emberjs.com/api/data/classes/DS.Store.html#method_query)会发起一个带有查询参数的GET请求，返回值跟findAll一样是一个 `PromiseArray`.

找出所有名字叫`Peter`的`person` models :

```javascript
// GET to /persons?filter[name]=Peter
this.store.query('person', { filter: { name: 'Peter' } }).then(function(peters) {
  // Do something with `peters`
});
```

### 查询单个Record

[`store.queryRecord()`](http://emberjs.com/api/data/classes/DS.Store.html#method_queryRecord)返回一个promise。

查询邮箱是'tomster@example.com' 的person model:
`tomster@example.com`:

```javascript
// GET to /persons?filter[email]=tomster@example.com
this.store.queryRecord('person', { filter: { email: 'tomster@example.com' } }).then(function(tomster) {
  // do something with `tomster`
});
```

创建、更新与删除
---
## 创建

[`createRecord()`](http://emberjs.com/api/data/classes/DS.Store.html#method_createRecord)

```js
store.createRecord('post', {
  title: 'Rails is Omakase',
  body: 'Lorem ipsum'
});
```

在controller或者route中直接调用 `this.store`就可以获得store对象.

## 更新

```js
this.store.findRecord('person', 1).then(function(tyrion) {
  // ...after the record has loaded
  tyrion.set('firstName', "Yollo");
});
```
Ember中修改属性特别便捷。例如，你可以使用 `Ember.Object`的
[`incrementProperty`](http://emberjs.com/api/classes/Ember.Object.html#method_incrementProperty) helper:

```js
person.incrementProperty('age'); // Happy birthday!
```

## 持久化

执行Model的任意实例的[`save()`](http://emberjs.com/api/data/classes/DS.Model.html#method_save)方法都会触发网络请求.


Ember Data会跟踪每个record的状态。这样当存储新创建的record和已经存在的record的时候就能区别对待。

Ember Data默认`POST` 新创建的record到它的type对应的url.

```javascript
var post = store.createRecord('post', {
  title: 'Rails is Omakase',
  body: 'Lorem ipsum'
});

post.save(); // => POST to '/posts'
```

已经存在的record使用HTTP的 `PATCH`动作.


```javascript
store.findRecord('post', 1).then(function(post) {
  post.get('title'); // => "Rails is Omakase"

  post.set('title', 'A new post');

  post.save(); // => PATCH to '/posts/1'
});
```
你可以通过record的
[`hasDirtyAttributes`](http://emberjs.com/api/data/classes/DS.Model.html#property_hasDirtyAttributes)
属性来辨别是否有没有提交的修改. 也可以使用
[`changedAttributes()`](http://emberjs.com/api/data/classes/DS.Model.html#method_changedAttributes)
方法辨别，哪些属性修改了，哪些没有. `changedAttributes`返回一个对象，key是被修改的属性名称，值是`[oldValue, newValue]`.

```js
person.get('isAdmin');            //=> false
person.get('hasDirtyAttributes'); //=> false
person.set('isAdmin', true);
person.get('hasDirtyAttributes'); //=> true
person.changedAttributes();       //=> { isAdmin: [false, true] }
```

这时，你可以调用save()方法持久化这些改变，也可以调用
[`rollbackAttributes()`](http://emberjs.com/api/data/classes/DS.Model.html#method_rollbackAttributes)
回滚这些修改。如果这个record `isNew`【是新建的】，那么就会被从store中删除.

```js
person.get('hasDirtyAttributes'); //=> true
person.changedAttributes();       //=> { isAdmin: [false, true] }

person.rollbackAttributes();

person.get('hasDirtyAttributes'); //=> false
person.get('isAdmin');            //=> false
person.changedAttributes();       //=> {}
```

## 验证错误

如果后端server尝试save的时候返回了验证错误，那么会体现在你的model的`errors`属性上。你可以通过下面的方法在你的模板中展示这些错误：

```hbs
{{#each post.errors.title as |error|}}
  <div class="error">{{error.message}}</div>
{{/each}}
{{#each post.errors.body as |error|}}
  <div class="error">{{error.message}}</div>
{{/each}}
```

## Promises

[`save()`](http://emberjs.com/api/data/classes/DS.Model.html#method_save) 返回一个promise。下面是典型操作:

```javascript
var post = store.createRecord('post', {
  title: 'Rails is Omakase',
  body: 'Lorem ipsum'
});

var self = this;

function transitionToPost(post) {
  self.transitionToRoute('posts.show', post);
}

function failure(reason) {
  // handle the error
}

post.save().then(transitionToPost).catch(failure);

// => POST to '/posts'
// => transitioning to posts.show route
```

## 删除

删除跟创建一样直接。 调用一个实例的[`deleteRecord()`](http://emberjs.com/api/data/classes/DS.Model.html#method_deleteRecord)方法. 之后会标记record为 `isDeleted`. 使用`save()`持久化删除操作. 或者使用`destroyRecord` 方法直接物理删除.

```js
store.findRecord('post', 1).then(function(post) {
  post.deleteRecord();
  post.get('isDeleted'); // => true
  post.save(); // => DELETE to /posts/1
});

// OR
store.findRecord('post', 2).then(function(post) {
  post.destroyRecord(); // => DELETE to /posts/2
});
```

事先推送record进入store
---
controller或者route请求store中没有的store时，store还要向adapter发起网络请求。
除了等待app请求一个record，你也可以提前向store中push record。

如果你知道你的用户下一步会使用什么record，那这就很有用。当他们请求的时候，会飞速返回。Ember可以自动重新渲染模板。

另外一个用例是与后端建立streaming connection的时候。当一个record被创建或者修改后，你应该马上更新UI。

### 推送

调用store的 [`push()`](http://emberjs.com/api/data/classes/DS.Store.html#method_push) 方法.

假设当app刚启动的时候，我们要预加载一些数据到store中。

我们使用`route:application`来做这项工作. `route:application` 是route层级结构中的最顶级route，他的 `model` hook 只在app启动的时候被调用一次.

```app/models/album.js
import Model from 'ember-data/model';
import attr from 'ember-data/attr';

export default Model.extend({
  title: attr(),
  artist: attr(),
  songCount: attr()
});
```

```app/routes/application.js
export default Ember.Route.extend({
  model() {
    this.store.push({
      data: [{
        id: 1,
        type: 'album',
        attributes: {
          title: 'Fewer Moving Parts',
          artist: 'David Bazan',
          songCount: 10
        },
        relationships: {}
      }, {
        id: 2,
        type: 'album',
        attributes: {
          title: 'Calgary b/w I Can\'t Make You Love Me/Nick Of Time',
          artist: 'Bon Iver',
          songCount: 2
        },
        relationships: {}
      }]
    });
  }
});
```

store的 `push()` 方法是一个低级API，接收JSON API文档，使用JSONAPISerializer进行解析 。JSON API终端 type名称必须跟model的严格对应。（这个例子里就是`album`，因为model就是`app/models/album.js`）。属性和关系的大小写也必须和model定义中完全一致。

如果你想在推送进store前标准化model默认的serializer，可使用
[`store.pushPayload()`](http://emberjs.com/api/data/classes/DS.Store.html#method_pushPayload) 方法.

```app/serializers/album.js
import RestSerializer from 'ember-data/serializers/rest';

export default RestSerializer.extend({
  normalize(typeHash, hash) {
    hash['songCount'] = hash['song_count']
    delete hash['song_count']
    return this._super(typeHash, hash);
  }

})
```

```app/routes/application.js
export default Ember.Route.extend({
  model() {
    this.store.pushPayload({
      albums: [
        {
          id: 1,
          title: 'Fever Moving Parts',
          artist: 'David Bazan',
          song_count: 10
        },
        {
          id: 2,
          title: 'Calgary b/w I Can\'t Make You Love Me/Nick Of Time',
          artist: 'Bon Iver',
          song_count: 2
        }
      ]
    });
  }
});
```

`push()` 方法在于复杂终端协同的时候也很重要。你的应用终端有时会执行一些业务逻辑，创建多个record。这对于Ember Data的`save()` API并不是完全对应，后者只能操作单条record。这是，你应该使用自己的ajax请求推送结果数据到store中，供其他UI页面使用。


```app/routes/confirm-payment.js
export default Ember.Route.extend({
  actions: {
    confirm: function(data) {
      $.ajax({
        data: data,
        method: 'POST',
        url: 'process-payment'
      }).then((digitalInventory) => {
        this.store.pushPayload(digitalInventory);
        this.transitionTo('thank-you');
      });
    }
  }
});
```

