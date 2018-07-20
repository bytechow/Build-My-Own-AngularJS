### 允许对被注入的函数添加本地变量（Providing Locals to Injected Functions）

有时你可能希望在注入时提供一些额外的参数（locals），这可能发生在两种情景中：你希望覆盖某些参数，或者某些参数还未曾在注射器中注册。

> 之后我们在介绍指令编译器时，就会用这个特性来实现对组件控制器注入 $scope、$element 和 $attrs 的功能

为了支持该特性，我们会为 injector.invoke 提供可选的第三个参数，这个参数是一个对象，其属性名和属性值，分别对应依赖名称和要替换的值。如果调用时传入了该参数，我们会先从这个对象中查找依赖，找不到时再从注射器缓存中查找，这跟我们之前介绍的 $scope.$eval 方法很相似

test/injector\_spec.js

```js
it('overrides dependencies with locals when invoking', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.constant('b', 2);
  var injector = createInjector(['myModule']);
  
  var fn = function(one, two) {
    return one + two;
  };
  fn.$inject = ['a', 'b'];

  expect(injector.invoke(fn, undefned, {
    b: 3
  })).toBe(4);
});
```

在源码实现中，同样地，当查找依赖时我们会先从 locals 中进行检索，若找到直接返回 locals 中检索到的值，没有找到才从 cache 中进行查找：

src/injector.js

```js
function invoke(fn, self, locals) {
  var args = _.map(fn.$inject, function(token) {
    if (_.isString(token)) {
      return locals && locals.hasOwnProperty(token) ?
        locals[token] :
        cache[token];
    } else {
      throw 'Incorrect injection token! Expected a string, got ' + token;
    }
  });
  return fn.apply(self, args);
}
```



