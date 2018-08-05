### 注入 $provider（Injecting $provider）

injector 实例对象可以让你获取已经配置好的依赖，但你没办法对依赖进行配置，也就是说这是一个只读的API。如果你希望配置依赖，你需要使用 $provder 依赖。

通过注入的 $provider，我们可以调用之前任务队列的生成方法来进行一些前置的配置。比如，我们可以在 provider 构造函数中注册一个常量：

test/injector\_spec.js

```js
it('allows injecting the $provide service to providers', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider($provide) {
    $provide.constant('b', 2);
    this.$get = function(b) {
      return 1 + b;
    };
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(3);
});
```

要注意的是，$provider 只是在 provider 构造函数中可用，运行时中不能再使用 $provder 进行配置或者增加依赖项：

test.injector\_spec.js

```js
it('does not allow injecting the $provide service to $get', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider() {
    this.$get = function($provide) {};
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.get('a');
  }).toThrow();
});
```

实际上，我们要注入的 $provider 就是我们在 createInjector 中声明的内部函数 $provider，它目前拥有 constant 和 provider 方法。所以，我们现在知道为什么叫这个名字了（[转至关联处](/modules-and-dependency-injection/11-modules-and-the-injector/11-07-registering-a-constant.md)）。现在我们只需要把 $provider 这个内部函数放到 provIderCache 中即可，就像我们之前加入 $injector 一样：

src/injector.js

```js
providerCache.$provide = {
  constant: function(key, value) {
    if (key === 'hasOwnProperty') {
      throw 'hasOwnProperty is not a valid constant name!';
    }
    providerCache[key] = value;
    instanceCache[key] = value;
  },
  provider: function(key, provider) {
    if (_.isFunction(provider)) {
      provider = providerInjector.instantiate(provider);
    }
    providerCache[key + 'Provider'] = provider;
  }
};
```

当然，我们的调用处也需要更新：

src/injector.js

```js
_.forEach(modulesToLoad, function loadModule(moduleName) {
  if (!loadedModules.hasOwnProperty(moduleName)) {
    loadedModules[moduleName] = true;
    var module = window.angular.module(moduleName);
    _.forEach(module.requires, loadModule);
    _.forEach(module._invokeQueue, function(invokeArgs) {
      var method = invokeArgs[0];
      var args = invokeArgs[1];
      providerCache.$provide[method].apply(providerCache.$provide, args);
    });
  }
});
```

通过提供 $injector 和 $provider，就可以让应用开发者直接调用 injector 的一些内部方法。虽然，大部分应用开发者只会用到模块和依赖注入两个特性，但这两个 API 让我们可以有进行配置和“临时”引入的便利方法。
