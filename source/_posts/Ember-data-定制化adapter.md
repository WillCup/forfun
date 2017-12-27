
title: Ember-data-定制化adapter
date: 2016-12-05 17:09:13
tags: [youdaonote]
---


adapter在Ember Data中决定数据怎样被持久化到后端数据存储层。比如处理URL格式、REST API的header等。(数据格式决定于
[serializer](../customizing-serializers/).)Ember Data的默认adapter有一些内置的假设规范，参看：REST API should look。如果你的后端不满足这些规范，就集成默认的adapter，修改一下它的方法。

一些典型的应用场景是在你的urls中使用
`underscores_case` , 使用中介而不是直接使用你后端的server，甚至直接使用
[local storage backend](https://github.com/locks/ember-localstorage-adapter).

继承adapter在Ember Data中是很轻松的过程。

如果你的后端你有一致性要求，你可以定义一个
`adapter:application`.  `adapter:application` 的优先级高于所有其他的默认Adapter, 但是低于model内指定的Adapters.

```app/adapters/application.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';

export default JSONAPIAdapter.extend({
  // Application specific overrides go here
});
```

如果你有一个model相对于其他的model在与后台沟通的时候需要额外的规则，那么你就可以创建一个Model指定的
Adapter：`ember generate adapter adapter-name`.
例如，运行`ember generate adapter post` 会创建下面的文件:

```app/adapters/post.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';

export default JSONAPIAdapter.extend({
  namespace: 'api/v1'
});
```

Ember Data默认有几个内置的adapter。

- [Adapter](http://emberjs.com/api/data/classes/DS.Adapter.html) 是最基础的adapter，不好喊任何功能。如果你要创建一个不同于其他的adapter功能的adapter，通常它是要给不错的起点。

- [JSONAPIAdapter](http://emberjs.com/api/data/classes/DS.JSONAPIAdapter.html)
`JSONAPIAdapter`是遵循默认JSON API规范的adapter，负责通过XHR传递JSON与http server沟通。

- [RESTAdapter](http://emberjs.com/api/data/classes/DS.RESTAdapter.html)
 `RESTAdapter` 通过XHR传递JSON允许你的store与http server直接沟通。2.0版本之前，这个是默认的adapter。


## 定制化JSONAPIAdapter

[JSONAPIAdapter](http://emberjs.com/api/data/classes/DS.JSONAPIAdapter.html)
有一些很好用的hooks来适应非标准的后台。

### URL规范

`JSONAPIAdapter`可以通过mode的名字来判断 URLs.比如，如果你通过ID查询`Post`:

```js
store.findRecord('post', 1).then(function(post) {
});
```

JSON API adapter会自动发送一个`GET` 请求到`/posts/1`.

JSON API adapter中的操作与url对应如下:

<table>
  <thead>
    <tr><th>Action</th><th>HTTP Verb</th><th>URL</th></tr>
  </thead>
  <tbody>
    <tr><th>Find</th><td>GET</td><td>/posts/123</td></tr>
    <tr><th>Find All</th><td>GET</td><td>/posts</td></tr>
    <tr><th>Update</th><td>PATCH</td><td>/posts/123</td></tr>
    <tr><th>Create</th><td>POST</td><td>/posts</td></tr>
    <tr><th>Delete</th><td>DELETE</td><td>/posts/123</td></tr>
  </tbody>
</table>

#### Pluralization Customization

To facilitate pluralizing model names when generating route urls Ember
Data comes bundled with
[Ember Inflector](https://github.com/stefanpenner/ember-inflector), a
ActiveSupport::Inflector compatible library for inflecting words
between plural and singular forms. Irregular or uncountable
pluralizations can be specified via `Ember.Inflector.inflector`.
A common way to do this is
（上面的没咋看懂，就是url中处理model负数问题的）
:

```app/app.js
// sets up Ember.Inflector
import './models/custom-inflector-rules';
```

```app/models/custom-inflector-rules.js
import Inflector from 'ember-inflector';

const inflector = Inflector.inflector;

inflector.irregular('formula', 'formulae');
inflector.uncountable('advice');

// Meet Ember Inspector's expectation of an export
export default {};
```

这告诉JSON API adapter对于`formula`的请求应该去`/formulae/1` 而不是 `/formulas/1`, 对于`advice`的请求应该去 `/advice/1` 而不是 `/advices/1`.

#### Endpoint Path Customization

`namespace` 属性可以作为指定url的前缀.

```app/adapters/application.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';

export default JSONAPIAdapter.extend({
  namespace: 'api/1'
});
```

请求`person` 被指向 `http://emberjs.com/api/1/people/1`.


#### Host Customization

默认adapter会向当前域发送请求。如果要指定新的域名需要设置adapter的`host`属性。

```app/adapters/application.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';

export default JSONAPIAdapter.extend({
  host: 'https://api.example.com'
});
```

请求`person` 会指向 `https://api.example.com/people/1`.


#### Path Customization

默认 `JSONAPIAdapter`会处理model的名字为复数，生成path。如果这并不适用于你的后台，那就override `pathForType` 方法.

例如，如果你不想复数化你的model名称，而是要把你的model名字从驼峰转为下滑线，你可以这边写
`pathForType` 方法:

```app/adapters/application.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';

export default JSONAPIAdapter.extend({
  pathForType: function(type) {
    return Ember.String.underscore(type);
  }
});
```

请求 `person` 被定向到 `/person/1`.
请求 `user-profile`定向到 `/user_profile/1`.

#### Headers customization

又的API必须要有一些header才行，如果提供一个key。在`JSONAPIAdapter`的 `headers`对象中我们可以设置任意的header信息。Ember Data会在发送每个ajax 请求的时候带上它们.

```app/adapters/application.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';

export default JSONAPIAdapter.extend({
  headers: {
    'API_KEY': 'secret key',
    'ANOTHER_HEADER': 'Some header value'
  }
});
```

`headers`也可以使用 computed property来支持动态header. 下面例子中是基于`session`service中的`authToken`属性的.

```app/adapters/application.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';

export default JSONAPIAdapter.extend({
  session: Ember.inject.service('session'),
  headers: Ember.computed('session.authToken', function() {
    return {
      'API_KEY': this.get('session.authToken'),
      'ANOTHER_HEADER': 'Some header value'
    };
  })
});
```

In some cases, your dynamic headers may require data from some
object outside of Ember's observer system (for example
`document.cookie`). You can use the
[volatile](http://emberjs.com/api/classes/Ember.ComputedProperty.html#method_volatile)
function to set the property into a non-cached mode causing the headers to
be recomputed with every request.

```app/adapters/application.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';

export default JSONAPIAdapter.extend({
  headers: Ember.computed(function() {
    return {
      'API_KEY': Ember.get(document.cookie.match(/apiKey\=([^;]*)/), '1'),
      'ANOTHER_HEADER': 'Some header value'
    };
  }).volatile()
});
```

#### Authoring Adapters

`defaultSerializer` 可以用来指定adapter的serializer. 只有model需要指定的serializer或者么有定义`serializer:application`的时候才会用到。

在app中, 指定
`serializer:application`通常很容易. 但是，如果你是一个社区adapter的作者，记住一定要设置这个属性，以免你的用户忘记没有指定这个属性，Ember就报错了。.

```app/adapters/my-custom-adapter.js
import JSONAPIAdapter from 'ember-data/adapters/json-api';

export default JSONAPIAdapter.extend({
  defaultSerializer: '-default'
});
```

## 社区的Adapters

可以从下面的地址中找到社区贡献的adapter，或许有适合你的哦:

- [Ember Observer](http://emberobserver.com/categories/data)
- [GitHub](https://github.com/search?q=ember+data+adapter&ref=cmdform)
- [Bower](http://bower.io/search/?q=ember-data-)

