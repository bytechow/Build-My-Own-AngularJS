### value 服务（Values）

value 服务跟 constant 服务有点类似。它在注册时的依赖体是一个值，而不是一个生产依赖的函数：

```js
it('allows registering a value', function() {
  var module = window.angular.module('myModule', []);
  module.value('a', 42);
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(42);
});
```

和 constant 服务不同的是，value 服务是不可以注入到 provider 或者 config 中去的，它们只可以在注入 instance 的地方被使用：

```js
it('does not make values available to confg blocks', function() {
  var module = window.angular.module('myModule', []);

  module.value('a', 42);
  module.confg(function(a) {});

  expect(function() {
    createInjector(['myModule']);
  }).toThrow();
});
```

首先，我们同样需要在模块加载器中加入 value 服务任务队列：

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: invokeLater('$provide', 'constant', 'unshift'),
  provider: invokeLater('$provide', 'provider'),
  factory: invokeLater('$provide', 'factory'),
  value: invokeLater('$provide', 'value'),
  confg: invokeLater('$injector', 'invoke', 'push', confgBlocks),
  run: function(fn) {
    moduleInstance._runBlocks.push(fn);
    return moduleInstance;
  },
  _invokeQueue: invokeQueue,
  _confgBlocks: confgBlocks,
  _runBlocks: []
};
```

value 服务的实现也是非常简单，我们会借用 factory 方法，将 value 服务对应的值，作为 factory 服务函数的返回值：

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
  },
  factory: function(key, factoryFn) {
    this.provider(key, {
      $get: enforceReturnValue(factoryFn)
    });
  },
  value: function(key, value) {
    this.factory(key, _.constant(value));
  }
};
```

value 服务的返回值也可以是 undefined，但根据我们之前的实现，这种情况会抛出异常：

```js
it('allows an undefned value', function() {
  var module = window.angular.module('myModule', []);
  module.value('a', undefned);
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBeUndefned();
});
```

所以，返回值检测应该是可选的，为此我们会为 $provider.factory 增加第三个参数，用于控制是否进行返回值检测：

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
  },
  factory: function(key, factoryFn, enforce) {
    this.provider(key, {
      $get: enforce === false ? factoryFn : enforceReturnValue(factoryFn)
    });
  },
  value: function(key, value) {
    this.factory(key, _.constant(value), false);
  }
};
```

如果服务是 value 服务，我们将会跳过返回值检测。

所以我们可以看到，value 基于 factory 实现，而 factory 基于 provider 进行实现。目前这样看来，value 服务好像真的和 constant 并没有差别，最终都会把依赖值放到 instance 缓存中。后面我们介绍 decoration（装饰器）时，你会看到 value 服务的优势。