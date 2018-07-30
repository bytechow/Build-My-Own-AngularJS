### 两个注射器：Provider 注射器和实例注射器

这两个注射器之间的第一个区别，在于 provider 注射器可以注入其他 provider：

test/injector\_spec.js

```js
it('injects another provider to a provider constructor function', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider() {
    var value = 1;
    this.setValue = function(v) {
      value = v;
    };
    this.$get = function() {
      return value;
    };
  });
  module.provider('b', function BProvider(aProvider) {
    aProvider.setValue(2);
    this.$get = function() {};
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(2);
});
```

此前我们注入的都是实例依赖，比如常量或 $get 方法的返回值。现在我们将会注入一个 provider，bProvider 依赖于 aProvider，所以需要注入 aProvider。然后在 bProvider 函数中使用 setValue 方法对 aProvider 进行配置。我们只需要对注入的 a 取值，就可以发现是否配置成功。

有一个最简单的方法可以满足上述测试用例：

src/injector.js

```js
function getService(name) {
  if (instanceCache.hasOwnProperty(name)) {
    if (instanceCache[name] === INSTANTIATING) {
      throw new Error('Circular dependency found: ' +
        name + ' <- ' + path.join(' <- '));
    }
    return instanceCache[name];
  } else if (providerCache.hasOwnProperty(name)) {
    return providerCache[name];
  } else if (providerCache.hasOwnProperty(name + 'Provider')) {
    path.unshift(name);
    instanceCache[name] = INSTANTIATING;
    try {
      var provider = providerCache[name + 'Provider'];
      var instance = instanceCache[name] = invoke(provider.$get);
      return instance;
    }
    fnally {
      path.shift();
      if (instanceCache[name] === INSTANTIATING) {
        delete instanceCache[name];
      }
    }
  }
}
```

现在



