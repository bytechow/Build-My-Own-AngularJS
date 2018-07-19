### 依赖注入（Dependency Injection）

我们现在已经拥有一个注射器的半成品，这个半成品可以加载模块、生成依赖缓存并查找已注册的依赖。但注射器的真正目标是依赖注入。依赖注入具体操作是调用组件函数，并自动找到所需的依赖，最后生成结果值。本章我们将介绍注射器的依赖注入。

依赖注入的基本思想是：我们会为 injector 增加一个方法，这个方法会对组件函数进行调用，找到函数参数对应的依赖，并注入到该函数中。

那么，注射器是怎么找到组件函数所需的实参是什么呢？最直接的方法就是通过组件函数的 $inject 属性显式地指定依赖项，这个属性是一个包含依赖项 key 值的数组。注射器会通过 key 值找到当前函数依赖的 value 值，value 值就成为组件函数的实参：

test/injector\_spec.js

```js
it('invokes an annotated function with dependency injection', function() {
  var module = window.angular.module('myModule', []);
  module.constant('a', 1);
  module.constant('b', 2);
  var injector = createInjector(['myModule']);

  var fn = function(one, two) {
    return one + two;
  };
  fn.$inject = ['a', 'b'];
  
  expect(injector.invoke(fn)).toBe(3);
});
```

当前我们只需要从注射器的 cache 中根据 key 找到依赖值即可：

src/loader.js

```js
function createInjector(modulesToLoad) {
  var cache = {};
  var loadedModules = {};
  var $provide = {
    constant: function(key, value) {
      if (key === 'hasOwnProperty') {
        throw 'hasOwnProperty is not a valid constant name!';
      }
      cache[key] = value;
    }
  };

  function invoke(fn) {
    var args = _.map(fn.$inject, function(token) {
      return cache[token];
    });
    return fn.apply(null, args);
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
  });
  
  return {
    has: function(key) {
      return cache.hasOwnProperty(key);
    },
    get: function(key) {
      return cache[key];
    },
    invoke: invoke
  };
}
```



