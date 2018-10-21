### factory 服务（Factories）

本节开始，我们将会实现 Angular 应用开发者最常用的高阶组件注册方法： factory、value 和 service。实际上，现在开发这三个服务并不费事，因为我们已经在前面的章节打下了基础。

首先，我们会先实现 factory 服务。简单来说，factory 就是一个生成依赖的函数。下面我们会建立一个测试用例，我们会注册一个返回数字 42 的 factory 服务函数，当我们获取这个依赖时，我们会得到值为42:

```js
it('allows registering a factory', function() {
  var module = window.angular.module('myModule', []);
  module.factory('a', function() { return 42; });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(42);
});
```

factory 函数允许注入依赖，（是通过 constant 加入的依赖？）。注意，它注入的是 instance 依赖。下面用例中的 bFactory 依赖 aFactory：

```js
it('injects a factory function with instances', function() {
  var module = window.angular.module('myModule', []);

  module.factory('a', function() { return 1; });
  module.factory('b', function(a) { return a + 2; });

  var injector = createInjector(['myModule']);

  expect(injector.get('b')).toBe(3);
});
```

跟 provider 类似，factory 服务产生的依赖都是单例的。也就是说，我们希望 factory 服务函数只会被调用一次，函数生产出来的依赖将会在我们之后每次引用该依赖时被注入，保证我们每次引入的值都是同一个：

```js
it('only calls a factory function once', function() {
  var module = window.angular.module('myModule', []);
  module.factory('a', function() { return {}; });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(injector.get('a'));
});
```

要实现 factory，我们需要先在模块加载器中将 factory 组件注册任务加入到任务队列中：

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: invokeLater('$provide', 'constant', 'unshift'),
  provider: invokeLater('$provide', 'provider'),
  factory: invokeLater('$provide', 'factory'),
  confg: invokeLater('$injector', 'invoke', 'push', confgBlocks),
  run: function(fn) {
    moduleInstance._runBlocks.push(fn);
    return moduleInstance;
  },
  _invokeQueue: invokeQueue,
  _confgBlocks: confgBlocks
  _runBlocks: []
};
```

然后我们需要在 injector 中加入对应的 $provider 方法，但这个方法具体要做些什么呢？其实，factory 服务函数的功能与 provider 的 $get 方法并没有差别，两者都生产依赖、允许注入依赖并且最多只能被调用一次。

实际上，facotry 服务确实就是对 provider 服务的封装，在内部，factory 服务函数会直接作为 provider 的 $get 方法使用：

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
      $get: factoryFn
    });
  }
};
```

注册一个 factory，也就意味着有一个 provider 会被创建，这个 provider 的 $get 方法就是 factory 注册时的函数。现在，我们开始真正感受了 provider 的强大之处，有了 provider，我们实现 factory 服务不费吹灰之力。

要注意的是，factory 服务没有提供对依赖进行预配置的功能。当你注册了 aFactory 并要使用该服务对应的依赖时，你需要获取 aProvider 并利用其 $get 方法生成依赖，这个过程中，你无法通过某种方法对依赖的生产过程产生影响。在此之前，你无法访问 aProvider，更不要谈对其进行配置。

我们还需要对 factory 做一个错误处理。我们需要确保 factory 服务函数会返回一个有效值，这个检测措施可以避免当应用代码出现 bug 时难以定位错误的问题：

```js
it('forces a factory to return a value', function() {
  var module = window.angular.module('myModule', []);
  module.factory('a', function() {});
  module.factory('b', function() {
    return null;
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.get('a');
  }).toThrow();
  expect(injector.get('b')).toBeNull();
});
```

该单元用例检测了两种 factory 服务函数返回值，分别为 undefined 和 null。如果返回值为 undefined，我们会抛出错误；而如果是 null，就按正常情况处理。

为了对函数的返回值进行检测，我们可以在外面再包裹一个函数，在函数内进行 factory 服务函数的执行和检测：

```js
function enforceReturnValue(factoryFn) {
  return function() {
    var value = instanceInjector.invoke(factoryFn);
    if (_.isUndefned(value)) {
      throw 'factory must return a value';
    }
    return value;
  };
}
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
  }
};
```

现在 factory 注册后，其生成的 provider 的 $get 是一个匿名函数，在匿名函数内部会使用 instanceInjector 对 factory 服务函数进行调用，并获取其返回值，如果检测返回值为 undefined，则会抛出一个明确的错误。

