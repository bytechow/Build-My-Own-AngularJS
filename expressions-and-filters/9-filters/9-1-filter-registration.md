### 注册过滤器
#### Filter Registration

首先要说明的是，过滤器服务允许注册过滤器。我们把过滤器名称和工厂函数作为注册函数的参数进行调用，就可以完成过滤器的注册。工厂函数将会返回实际的过滤器。之后，我们可以使用 `filter` 函数来获取已注册的过滤器。我们会把相关的测试放到一个新的测试文件中去：

_test/filter_spec.js_

```js
'use strict';

var register = require('../src/filter').register;
var filter = require('../src/filter').filter;

describe('filter', function() {

  it('can be registered and obtained', function() {
    var myFilter = function() {};
    var myFilterFactory = function() {
      return myFilter;
    };
    register('my', myFilterFactory);
    expect(filter('my')).toBe(myFilter);
  });
  
});
```

我们现在就可以直接实现这两个函数。这两个函数都可以访问到同一个存储对象，这个对象的键是过滤器的名称，值是过滤器。当我们注册一个过滤器时，值会被替换为这个工厂函数的返回值。实现这部分逻辑的代码也会放到一个新文件中：

_src/filter.js_

```js
'use strict';

var filters = {};

function register(name, factory) {
  var filter = factory();
  filters[name] = filter;
  return filter;
}

function filter(name) {
  return filters[name];
}

module.exports = { register: register, filter: filter };
```

注册函数还支持使用简写形式一次性注册多个过滤器。我们只需要传递一个对象即可，这个对象的键是过滤器名称，值是名称对应的过滤器工厂函数：

_test/filter_spec.js_

```js
it('allows registering multiple filters with an object', function() {
  var myFilter = function() {};
  var myOtherFilter = function() {};
  register({
    my: function() {
      return myFilter;
    },
    myOther: function() {
      return myOtherFilter;
    }
  });
  
  expect(filter('my')).toBe(myFilter);
  expect(filter('myOther')).toBe(myOtherFilter);
});
```

在实际开发中，只要我们发现第一个参数是对象，就对对象中的每一个键值对递归调用 `register` 方法：

_src/filter.js_

```js
'use strict';

var _ = require('lodash');

var filters = {};

function register(name, factory) {
  if (_.isObject(name)) {
    return _.map(name, function(factory, name) {
      return register(name, factory);
    });
  } else {
    var filter = factory();
    filters[name] = filter;
    return filter;
  }
}

function filter(name) {
  return filters[name];
}

module.exports = { register: register, filter: filter };
```

这样我们就完成了过滤器服务的注册流程。如前所述，一旦我们实现了依赖注入系统，就会对这个注册流程进行调整。