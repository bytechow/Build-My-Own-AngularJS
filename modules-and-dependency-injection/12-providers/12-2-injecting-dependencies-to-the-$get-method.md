### 对 $get 方法注入依赖（Injecting Dependencies To The $get Method）

现在我们仍然对于为什么要引入 provider 机制感到不解，其实只是我们还没用利用到 provider 可以注入依赖到 $get 的特性。

现在我们考虑一种情况——应用组件拥有它自己的依赖，在这种情况下，$get 就能派上用场了。实际上，这种情况就是我们前面提到的依赖中包含其他依赖的情况。针对这一情况，现在我们就可以通过对 $get 进行依赖注入来满足：

test/injector\_spec.js

```js
it('injects the $get method of a provider', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.provider('b', {
    $get: function(a) {
      return a + 2;
    }
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('b')).toBe(3);
});
```

上面的用例需求比较简单，我们可以通过上一章节实现的 invoke 对 $get 方法进行调用即可。

src/injector.js

```js
var $provide = {
  constant: function(key, value) {
    if (key === 'hasOwnProperty') {
      throw 'hasOwnProperty is not a valid constant name!';
    }
    cache[key] = value;
  },
  provider: function(key, provider) {
    cache[key] = invoke(provider.$get, provider);
  }
};
```

注意，由于 $get 是属于 provider 对象的方法，所以我们在调用时要把 provider 对象作为上下文。

现在，我们可以知道使用 invoke 方法的场景有两个，一是对自定义函数进行依赖注入，其次是在 injector 内部对 provider.$get 进行注入。

