### 保证一切皆单例（Making Sure Everything Is A Singleton）

你可能听说过“ Angular 中一切皆单例 ”，这是正确的。当在两个不同的地方注入相同的依赖，最终获取到的是对同一个对象的引用。

但我们目前的实现还没做到完全的单例化，下面这个测试用例中：

test/injector\_spec.js

```js
it('instantiates a dependency only once', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', {
    $get: function() {
      return {};
    }
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(injector.get('a'));
});
```

在这个测试用例中，我们最终会调用 $get 两次，而每一次都会返回一个新对象，这与我们的需求不符。

解决方法就是在我们每次通过 provider 实例化依赖后，都把依赖放到实例依赖缓存中。以后再获取这个依赖时，由于我们首先检索实例缓存，找到之后就会直接返回，而不会再次调用 provider.$get 生成依赖：

src/injector.js

```js
function getService(name) {
  if (instanceCache.hasOwnProperty(name)) {
    return instanceCache[name];
  } else if (providerCache.hasOwnProperty(name + 'Provider')) {
    var provider = providerCache[name + 'Provider'];
    var instance = instanceCache[name] = invoke(provider.$get);
    return instance;
  }
}
```



