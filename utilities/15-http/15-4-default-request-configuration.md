### 默认请求配置（default-request-configuration）

正如我们所见，`$http`函数会接收一个请求配置对象作为它唯一的参数，但这个对象包含了要用于生成请求的所有属性，即URL、HTTP 方法和传输数据等。但并不是所有的属性都是必需指明的，所以我们对一个必要的属性提供了一个默认的配置。

我们要设置的第一个默认值是请求方法，默认为 GET 方法：

```js
it('uses GET method by default', function() {
  $http({
    url: 'http://teropa.info'
  });
  expect(requests.length).toBe(1);
  expect(requests[0].method).toBe('GET');
});
```

我们可以在`$http`中构建一个对象存放“默认配置”，并且让自定义配置对默认属性进行扩展（extend）,也就是自定义属性可以覆盖默认属性：

```js
return function $http(requestConfg) {
  var deferred = $q.defer();

  var confg = _.extend({
    method: 'GET'
  }, requestConfg);
  
  // ...
};
```

我们现在需要引入 LoDash 了：

```js
'use strict';

var _ = require('lodash');
```

并不是所有传入 $http 函数的参数都是可以预配置的，但在稍后的章节我们还会接触到几个可以被预配置的。