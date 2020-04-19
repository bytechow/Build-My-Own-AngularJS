### Filter 过滤器
#### The Filter Filter

现在过滤器已经实现了，接下来我们会用剩余的章节来介绍 Angular 内部实现的一个过滤器：`filter` 过滤器。

简单地说，filter 过滤器的目的是将表达式中使用到的数组筛选到某个子集中。你可以指定一个匹配规则来筛选数组中的元素，这时候表达式的结果就是一个元素全部符合匹配规则的新数组。这有点像是在数组中查找指定的元素集合。

filter 过滤器通常与 `ngRepeat` 指令一起使用，当你需要为数组中符合条件的元素（而不是全部元素）重复生成一段 DOM时，但这个过滤器并不局限于 `ngRepeat` 指令，只要表达式中有数组都可以使用：

我们先先建一个新文件，在里面加入一个断言，断言 filter 过滤器是真实存在的，也能通过 filter 服务获取到。

_test/filter_filter_spec.js_

```js
'use strict';

var filter = require('../src/filter').filter;

describe('filter filter', function() {

  it('is available', function() {
    expect(filter('filter')).toBeDefined();
  });

});
```

filter 过滤器的工厂函数会放到一个单独的源代码文件中：

_src/filter_filter.js_

```js
'use strict';

function filterFilter() {
  return function() {
    
  };
}

module.exports = filterFilter;
```

我们会在 `filter.js` 注册这个过滤器：

_src/filter.js_

```js
// 'use strict';

// var _ = require('lodash');

// var filters = {};

// function register(name, factory) {
//   if (_.isObject(name)) {
//     return _.map(name, function(factory, name) {
//       return register(name, factory);
//     });
//   } else {
//     var filter = factory();
//     filters[name] = filter;
//     return filter;
//   }
// }

// function filter(name) {
//   return filters[name];
// }

register('filter', require('./filter_filter'));

// module.exports = { register: register, filter: filter };
```

注册完以后，下面我们来看看 filter 过滤器具体能干什么。