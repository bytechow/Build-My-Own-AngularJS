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

可以看到，现在我们检索 providerCache 可能有两个不同的目的：查找缓存中的 provider 并生成依赖值，或者就是要获取 provider 本身。

但上面的解决方案太简单粗暴了。仔细想想，我们不可能将 provider 或者 instance 在任意地方注入。例如，你可以注入 provider 到一个 provider 构造函数中，但不可以注入 instance:

> 译者注: provider 构造函数没有懒加载，当其实例化时，其依赖的 instance 可能还没有被生成出来，所以 provider 构造函数不能注入 instance

test/injector\_spec.js

```js
it('does not inject an instance to a provider constructor function', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider() {
    this.$get = function() {
      return 1;
    };
  });
  module.provider('b', function BProvider(a) {
    this.$get = function() {
      return a;
    };
  });
  expect(function() {
    createInjector(['myModule']);
  }).toThrow();
});
```

当你使用 provider 构造函数时，就仅允许注入 provider，而非 instance。

相反，当你使用 $get、injector.invoke\(\)、injector.get\(\)，就不允许注入 provider 构造函数。

test/injector.js

```js
it('does not inject a provider to a $get function', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider() {
    this.$get = function() {
      return 1;
    };
  });
  module.provider('b', function BProvider() {
    this.$get = function(aProvider) {
      return aProvider.$get();
    };
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.get('b');
  }).toThrow();
});
```

```js
it('does not inject a provider to invoke', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider() {
    this.$get = function() {
      return 1;
    }
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.invoke(function(aProvider) {});
  }).toThrow();
});
```

```js
it('does not give access to providers through get', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', function AProvider() {
    this.$get = function() {
      return 1;
    };
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.get('aProvider');
  }).toThrow();
});
```

我们建立的这些测试都是为了让我们更好地区分两种依赖注入：provider 构造函数只能注入其他 provider，而 $get 方法和 injector 对外的接口只能注入实例。依赖实例可能是由 provider 生成的，但 provider 是不会对外暴露的。

我们可以通过将当前的单个 injector 分开成两个不同 injector 来实现这个目标：一个 injector 处理 provider，而另一个处理 instance。instanceInjector 将会对外暴露，而 providerInjector 只在 createInjector 内部使用。

下面我们会一步步地实现，而在最后我们会呈现 createInjector 的完整源代码。

首先我们会在 createInjector 内部创建一个内部方法 createInternalInjector，这个方法是用于创建上述的两个 injector 的。这个函数将会接受两个参数：要查找的依赖缓存空间（分别为 providerCache 和 instanceCache）、一个工厂函数（用于在指定的依赖缓存空间找不到依赖时的回退方案）。

src/injector.js

```js
function createInternalInjector(cache, factoryFn) {
}
```

我们需要把所有含有依赖查找逻辑的方法都迁移到 createInternalInjector 内部，因为我们要把查找范围都限定在某个缓存中。首先是搬迁 getService：

src/injector.js

```js
function createInternalInjector(cache, factoryFn) {
  function getService(name) {
    if (cache.hasOwnProperty(name)) {
      if (cache[name] === INSTANTIATING) {
        throw new Error('Circular dependency found: ' +
          name + ' <- ' + path.join(' <- '));
      }
      return cache[name];
    } else {
      path.unshift(name);
      cache[name] = INSTANTIATING;
      try {
        return (cache[name] = factoryFn(name));
      }
      fnally {
        path.shift();
        if (cache[name] === INSTANTIATING) {
          delete cache[name];
        }
      }
    }
  }
}
```

现在我们不会在 else 分支里面显式调用 provider。我们会利用 factoryFn 来处理依赖查找不到的情况。我们之后会看到它是如何工作的。

invoke 和 instantiate 都依赖 getService 方法，所以也要搬到 createInternalInjector 里面，但函数体内部的逻辑不需要更改。

src/injector.js

```js
function createInternalInjector(cache, factoryFn) {
  function getService(name) {
    if (cache.hasOwnProperty(name)) {
      if (cache[name] === INSTANTIATING) {
        throw new Error('Circular dependency found: ' +
          name + ' <- ' + path.join(' <- '));
      }
      return cache[name];
    } else {
      path.unshift(name);
      cache[name] = INSTANTIATING;
      try {
        return (cache[name] = factoryFn(name));
      }
      fnally {
        path.shift();
        if (cache[name] === INSTANTIATING) {
          delete cache[name];
        }
      }
    }
  }

  function invoke(fn, self, locals) {
    var args = annotate(fn).map(function(token) {
      if (_.isString(token)) {
        return locals && locals.hasOwnProperty(token) ?
          locals[token] :
          getService(token);
      } else {
        throw 'Incorrect injection token! Expected a string, got ' + token;
      }
    });
    if (_.isArray(fn)) {
      fn = _.last(fn);
    }
    return fn.apply(self, args);
  }

  function instantiate(Type, locals) {
    var instance = Object.create((_.isArray(Type) ? _.last(Type) : Type).prototype);
    invoke(Type, instance, locals);
    return instance;
  }
}
```

最后就是要返回的 injector 实例，基本与之前保持一致：

src/injector.js

```js
function createInternalInjector(cache, factoryFn) {
  // ...
  return {
    has: function(name) {
      return cache.hasOwnProperty(name) ||
        providerCache.hasOwnProperty(name + 'Provider');
    },
    get: getService,
    annotate: annotate,
    invoke: invoke,
    instantiate: instantiate
  };
}
```

除了以上三个函数需要搬迁，annotate 方法不需要变动。而 has 方法除了要查找对应的依赖缓存空间，也要查找 provider 缓存。

现在我们就可以利用 createInternalInjector 来创建两个注射器。providerInjecctor 负责处理 provider 缓存。它的工厂函数（factoryFn）将会抛出一个异常，告诉我们依赖查找不到：

src/injector.js

```js
function createInjector(modulesToLoad) {
  var providerCache = {};
  var providerInjector = createInternalInjector(providerCache, function() {
    throw 'Unknown provider: '+path.join(' <- ');
  });
  // ...
}
```

对于 instance injector，它的工厂函数会查找 provider 并生成依赖。这实际上就是之前我们在 getService 的 else 分支做的“简单方案”：

src/injector.js

```js
function createInjector(modulesToLoad) {
  var providerCache = {};
  var providerInjector = createInternalInjector(providerCache, function() {
    throw 'Unknown provider: ' + path.join(' <- ');
  });
  var instanceCache = {};
  var instanceInjector = createInternalInjector(instanceCache, function(name) {
    var provider = providerInjector.get(name + 'Provider');
    return instanceInjector.invoke(provider.$get, provider);
  });
  // ...
}
```

注意到了吗？我们使用 providerInjector 查找 provider，但是使用 instanceInjector.invoke 来调用 provider 的 $get 方法，我们这样可以保证只有实例会被注入到 $get 中。

另外，我们会使用 providerInjector.instantiate 来实例化 provider 构造函数，这样我们就能保证我们只能注入其他 provider:

src/injector.js

```js
provider: function(key, provider) {
  if (_.isFunction(provider)) {
    provider = providerInjector.instantiate(provider);
  }
  providerCache[key + 'Provider'] = provider;
}
```

常量是一个例外，我们应该能够在两个缓存中都能找到常量：

src/injector.js

```js
constant: function(key, value) {
  if (key === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid constant name!';
  }
  providerCache[key] = value;
  instanceCache[key] = value;
},
```

最后，我们会返回 instanceInjector 作为 createInjector 的结果值。

下面是 createInjector 的全部代码：

src/injector.js

```js
function createInjector(modulesToLoad, strictDi) {
  var providerCache = {};
  var providerInjector = createInternalInjector(providerCache, function() {
    throw 'Unknown provider: ' + path.join(' <- ');
  });
  var instanceCache = {};
  var instanceInjector = createInternalInjector(instanceCache, function(name) {
    var provider = providerInjector.get(name + 'Provider');
    return instanceInjector.invoke(provider.$get, provider);
  });
  var loadedModules = {};
  var path = [];
  strictDi = (strictDi === true);
  var $provide = {
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

  function annotate(fn) {
    if (_.isArray(fn)) {
      return fn.slice(0, fn.length - 1);
    } else if (fn.$inject) {
      return fn.$inject;
    } else if (!fn.length) {
      return [];
    } else {
      var source = fn.toString().replace(STRIP_COMMENTS, '');
      var argDeclaration = source.match(FN_ARGS);
      return _.map(argDeclaration[1].split(','), function(argName) {
        return argName.match(FN_ARG)[2];
      });
    }
  }

  function createInternalInjector(cache, factoryFn) {
    function getService(name) {
      if (cache.hasOwnProperty(name)) {
        if (cache[name] === INSTANTIATING) {
          throw new Error('Circular dependency found: ' +
            name + ' <- ' + path.join(' <- '));
        }
        return cache[name];
      } else {
        path.unshift(name);
        cache[name] = INSTANTIATING;
        try {
          return (cache[name] = factoryFn(name));
        }
        fnally {
          path.shift();
          if (cache[name] === INSTANTIATING) {
            delete cache[name]
          }
        }
      }
    }

    function invoke(fn, self, locals) {
      var args = _.map(annotate(fn), function(token) {
        if (_.isString(token)) {
          return locals && locals.hasOwnProperty(token) ?
            locals[token] :
            getService(token);
        } else {
          throw 'Incorrect injection token! Expected a string, got ' + token;
        }
      });
      if (_.isArray(fn)) {
        fn = _.last(fn);
      }
      return fn.apply(self, args);
    }

    function instantiate(Type, locals) {
      var UnwrappedType = _.isArray(Type) ? _.last(Type) : Type;
      var instance = Object.create(UnwrappedType.prototype);
      invoke(Type, instance, locals);
      return instance;
    }
    return {
      has: function(name) {
        return cache.hasOwnProperty(name) ||
          providerCache.hasOwnProperty(name + 'Provider');
      },
      get: getService,
      annotate: annotate,
      invoke: invoke,
      instantiate: instantiate
    };
  }
  _.forEach(modulesToLoad, function loadModule(moduleName) {
    if (!loadedModules.hasOwnProperty(moduleName)) {
      loadedModules[moduleName] = true;
      var module = window.angular.module(moduleName);
      _.forEach(module.requires, loadModule);
      _.forEach(module._invokeQueue, function(invokeArgs) {
        var method = invokeArgs[0];
        var args = invokeArgs[1];
        $provide[method].apply($provide, args);
      });
    }
  })
  return instanceInjector;
}
```

我们现在实现了依赖注入的两个阶段形态：

1. providerInjector 会在执行模块任务队列时被注册，之后就不会再发生变化
2. 在运行时，我们在调用注射器的API时，会同时实例化依赖，该过程会发生在 instanceInjecot（工厂函数） 中