### service 服务（Services）

factory 是一个普通的 JavaScript 函数，而 service 是一个构造函数。当你注册一个 service，其服务函数会被当作一个构造函数，之后在执行时就会执行这个函数，并生成一个实例。

```js
it('allows registering a service', function() {
  var module = window.angular.module('myModule', []);
  module.service('aService', function MyService() {
    this.getValue = function() { return 42; };
  });

  var injector = createInjector(['myModule']);
  expect(injector.get('aService').getValue()).toBe(42);
});
```

当然 service 服务函数可以注入依赖：

```js
it('injects service constructors with instances', function() {
  var module = window.angular.module('myModule', []);
  module.value('theValue', 42);
  module.service('aService', function MyService(theValue) {
    this.getValue = function() {
      return theValue;
    };
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('aService').getValue()).toBe(42);
});
```

当然按照 Angular 的规矩，service 也是单例的，其服务函数（构造函数）也只会被执行一次，执行结果会被缓存起来以供后续使用：

```js
it('only instantiates services once', function() {
  var module = window.angular.module('myModule', []);
  module.service('aService', function MyService() {});
  var injector = createInjector(['myModule']);
  expect(injector.get('aService')).toBe(injector.get('aService'));
});
```

要实现 service，也同样是先在模块加载器中加入队列方法：

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: invokeLater('$provide', 'constant', 'unshift'),
  provider: invokeLater('$provide', 'provider'),
  factory: invokeLater('$provide', 'factory'),
  value: invokeLater('$provide', 'value'),
  service: invokeLater('$provide', 'service'),
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

接着我们实现 $provider 里面的 service 方法，service 方法会接收依赖名称和构造函数，同时它也是借用了 factory 方法：

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
  },
  service: function(key, Constructor) {
    this.factory(key, function() {});
  }
};
```

我们要做的包括创建构造函数实例，也包括注入依赖。我们可以直接利用之前我们在第 9 章实现的 instantiate 的方法：

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
  },
  service: function(key, Constructor) {
    this.factory(key, function() {
      return instanceInjector.instantiate(Constructor);
    });
  }
};
```

现在我们可以看到 facotry、value 和 service 服务都是建立在 provider 的基础上，它们实际上都是 provider。只不过，这三个明确的 API 设计封装，能够让应用开发者对自己要建立什么样的 provider 服务类型更清晰，所以应用开发者会更倾向于使用这三个 API，而不是 provider 本身来注册服务。但我们还是可以在适当的时候使用 provider，毕竟已经是实现好了。