### 在注入时允许绑定this（Binding this in Injected Functions）

注入依赖的函数也有可能是对象里面的方法，既然是方法，那么就必须解决 this 上下文绑定的问题。在 JavaScript 中，如果直接调用对象方法，this 上下文会自动绑定，但我们现在是通过 injector.invoke 方法间接调用，所以必须手动解决绑定问题。我们可以通过为 injector.invoke 方法增加一个可选参数来解决，这个参数就是我们要求在依赖函数调用时绑定的上下文。

test/injector\_spec.js

```js
it('invokes a function with the given this context', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  var injector = createInjector(['myModule']);

  var obj = {
    two: 2,
    fn: function(one) {
      return one + this.two;
    }
  };
  obj.fn.$inject = ['a'];

  expect(injector.invoke(obj.fn, obj)).toBe(3);
});
```

我们可以利用 Function.prototype.apply 原型方法来进行强制绑定，只需要把之前传入的第一个参数从 null 变成上下文实参就可以了：

src/injector.js

```js
function invoke(fn, self) {
  var args = _.map(fn.$inject, function(token) {
    if (_.isString(token)) {
      return cache[token];
    } else {
      throw 'Incorrect injection token! Expected a string, got ' + token;
    }
  });
  return fn.apply(self, args);
}
```



