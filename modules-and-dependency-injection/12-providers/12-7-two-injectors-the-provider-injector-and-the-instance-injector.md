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

现在我们检索 providerCache 可能有两个不同的目的：查找缓存中的 provider 并生成依赖值，或者就是要获取 provider 本身。

但上面的解决方案太简单粗暴了。仔细想想，我们不可能将 provider 或者 instance 在任意地方注入。例如，你可以注入 provider 到一个 provider 构造函数中，但不可以注入 instance:

> 译者注: provider 构造函数没有懒加载，当其实例化时，其依赖的 instance 可能还没有被生成出来，所以 provider 构造函数不能注入 instance

test/injector_spec.js

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

相反，当你使用 $get、injector.invoke()、injector.get()，就不允许注入 provider。

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

我们建立的这些测试都是为了让我们更好地区分两种依赖注入：