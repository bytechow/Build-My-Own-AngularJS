decorator 服务（Decorators）

我们要实现的最后一个依赖注入特性是 decorator 装饰器。decorator 与之前实现的特性不同的是，我们不会使用 decorator 来注册依赖，而会用它来修改依赖。

decorator 这个名字是从面向对象的装饰器设计模式（Decorator design pattern）得来的。

decorator 有用之处在于，我们可以用来配置第三方依赖。我们可以注册针对某个依赖库或者 Angualr 本身的 decorator，就像 Brian Ford 说过一样。

让我们看看 decorator 是如何工作的。下面我们会创建一个 factory，然后使用 factory 的名称创建一个同名的装饰器。decorator 是一个函数，也可以像 factory 一样注入依赖，另外它还拥有一个特殊的可注入参数 $delegate。这个 $delegate 会被注入对应服务生成的依赖值。在下面的事例中，它就会被注入为名为 aValue 的 factory 所生成的依赖：

```js
it('allows changing an instance using a decorator', function() {
  var module = window.angular.module('myModule', []);
  module.factory('aValue', function() {
    return {
      aKey: 42
    };
  });
  module.decorator('aValue', function($delegate) {
    $delegate.decoratedKey = 43;
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('aValue').aKey).toBe(42);
  expect(injector.get('aValue').decoratedKey).toBe(43);
});
```

Angular 也允许对一个依赖生成多个装饰器，这些装饰器都会应用到该依赖上：

```js
it('allows multiple decorators per service', function() {
  var module = window.angular.module('myModule', []);
  module.factory('aValue', function() {
    return {};
  });
  module.decorator('aValue', function($delegate) {
    $delegate.decoratedKey = 42;
  });
  module.decorator('aValue', function($delegate) {
    $delegate.otherDecoratedKey = 43;
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('aValue').decoratedKey).toBe(42);
  expect(injector.get('aValue').otherDecoratedKey).toBe(43);
});
```

正如我们所讨论的，decorator 函数可以注入其他依赖：

```js
it('uses dependency injection with decorators', function() {
  var module = window.angular.module('myModule', []);
  module.factory('aValue', function() {
    return {};
  });
  module.constant('a', 42);
  module.decorator('aValue', function(a, $delegate) {
    $delegate.decoratedKey = a;
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('aValue').decoratedKey).toBe(42);
});
```

我们依然会在模块加载器中加入 decorator 的 API，并在 $provider 上面加入对应的任务队列执行方法，这个方法接收两个参数：要装饰的依赖名称和装饰器函数。

我们要做的是在依赖创建阶段加入一个钩子，为此我们首先要获取依赖的 provider：

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
  },
  decorator: function(serviceName, decoratorFn) {
    var provider = providerInjector.get(serviceName + 'Provider');
  }
};
```

当依赖被创建成功时，我们需要获取 provider 的返回值，并用某种方式对返回值进行更改。我们要做的是重写 provider 的 $get 方法：

```js
decorator: function(serviceName, decoratorFn) {
  var provider = providerInjector.get(serviceName + 'Provider');
  var original$get = provider.$get;
  provider.$get = function() {
    var instance = instanceInjector.invoke(original$get, provider);
    // Modifcations will be done here
    return instance;
  };
}
```

我们上面实现的是最基础的版本，仅仅重写了 provider.$get 方法，但并没有加入装饰器的处理逻辑。

最后一步自然是执行 decorator 函数。注意我们是通过依赖注入的方式来执行 decorator 函数，还加入 $delegate 这个额外参数。具体来说，我们使用了 instanceInjection 进行依赖注入，并且通过第 9 章实现的 locals 参数来增加额外变量：

```js
decorator: function(serviceName, decoratorFn) {
  var provider = providerInjector.get(serviceName + 'Provider');
  var original$get = provider.$get;
  provider.$get = function() {
    var instance = instanceInjector.invoke(original$get, provider);
    instanceInjector.invoke(decoratorFn, null, {
      $delegate: instance
    });
    return instance;
  };
}
```

最后，我们只需要把接口暴露出来就可以了：

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: invokeLater('$provide', 'constant', 'unshift'),
  provider: invokeLater('$provide', 'provider'),
  factory: invokeLater('$provide', 'factory'),
  value: invokeLater('$provide', 'value'),
  service: invokeLater('$provide', 'service'),
  decorator: invokeLater('$provide', 'decorator'),
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

实现 decorator 之后，我们终于开发完成了完整的 Angular 依赖注入功能。

