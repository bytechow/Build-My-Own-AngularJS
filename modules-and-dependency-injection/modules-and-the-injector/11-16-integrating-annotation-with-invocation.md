### 在调用过程中使用注解（Integrating Annotation with Invocation）

我们现在可以通过三种方式提取依赖名称：$inject，提取数组元素，过滤注射器函数源码。接下来，我们要做的是在 injector.invoke 中集成后面两种方式。首先是采用数组式注解的注射器函数，也就是行内式注入：

test/injector_spec.js

```js
it('invokes an array-annotated function with dependency injection', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.constant('b', 2);
  var injector = createInjector(['myModule']);

  var fn = ['a', 'b', function(one, two) {
    return one + two;
  }];

  expect(injector.invoke(fn)).toBe(3);
});
```

然后是采用无注解形式的注射器函数，也就是推断式注入：

test/injecto_spec.js

```js
it('invokes a non-annotated function with dependency injection', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.constant('b', 2);
  var injector = createInjector(['myModule']);

  var fn = function(a, b) {
    return a + b;
  };
  
  expect(injector.invoke(fn)).toBe(3);
});
```

在 invoke 方法中，我们需要做两处改动。首先，我们需要使用 annotate 方法代替直接使用 fn.$inject 获取依赖列表。然后，我们需要判断传入的 fn 参数是不是数组，是的话，我们就将最后一个数组元素作为注射器函数：

src/injector.js

```js
function invoke(fn, self, locals) {
  var args = _.map(annotate(fn), function(token) {
    if (_.isString(token)) {
      return locals && locals.hasOwnProperty(token) ?
        locals[token] :
        cache[token];
    } else {
      throw 'Incorrect injection token! Expected a string, got ' + token;
    }
  });
  if (_.isArray(fn)) {
    fn = _.last(fn);
  }
  return fn.apply(self, args);
}
```

现在，这三种注入方式我们都可以满足了！