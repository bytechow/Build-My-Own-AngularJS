### 拒绝不是字符串的依赖标记（Rejecting Non-String DI Token）

我们看到了 $inject 数组包含依赖名称。如果某人在 $inject 数组中加入一些非法的值（比如一个数字），目前程序会返回undefined，但我们应当用抛出异常的方式来提醒开发者。

test/injector\_spec.js

```js
it('does not accept non-strings as injection tokens', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  var injector = createInjector(['myModule']);

  var fn = function(one, two) {
    return one + two;
  };
  fn.$inject = ['a', 2];
  
  expect(function() {
    injector.invoke(fn);
  }).toThrow();
});
```

我们可以在检索依赖之前，对传入的 token 进行检查：

src/injector.js

```js
function invoke(fn) {
  var args = _.map(fn.$inject, function(token) {
    if (_.isString(token)) {
      return cache[token];
    } else {
      throw 'Incorrect injection token! Expected a string, got ' + token;
    }
  });
  return fn.apply(null, args);
}
```



