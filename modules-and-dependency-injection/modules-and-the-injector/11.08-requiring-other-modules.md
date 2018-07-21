### 引入其他模块（Requiring Other Modules）

我们已经创建过包含一个模块的注射器，在 Angular 中也允许我们创建包含多个模块的注射器。最直接的方法，就是在调用 createInjector 时提供多个模块名称作为参数。当然属于各个模块的应用组件都会被注册：

test/injector\_spec.js

```js
it('loads multiple modules', function() {
  var module1 = window.angular.module('myModule', []);
  var module2 = window.angular.module('myOtherModule', []);
  module1.constant('aConstant', 42);
  module2.constant('anotherConstant', 43);
  var injector = createInjector(['myModule', 'myOtherModule']);

  expect(injector.has('aConstant')).toBe(true);
  expect(injector.has('anotherConstant')).toBe(true);
});
```

目前已经是对所有加载的模块进行遍历，并依次执行对应模块的任务队列，所以当前代码已经满足了要求。

还有一种会加载多个模块的情况可能会出现，也就是模块依赖其他模块。我们目前注册模块用的是 angular.module，该方法的第二个参数就是用来注入其他模块的。当一个模块被加载后，它依赖的模块也会被加载：

test/injector\_spec.js

```js
it('loads the required modules of a module', function() {
  var module1 = window.angular.module('myModule', []);
  var module2 = window.angular.module('myOtherModule', ['myModule']);
  module1.constant('aConstant', 42);
  module2.constant('anotherConstant', 43);
  var injector = createInjector(['myOtherModule']);

  expect(injector.has('aConstant')).toBe(true);
  expect(injector.has('anotherConstant')).toBe(true);
});
```

还应该支持依赖传递，比如模块A是模块B的依赖，模块B是模块C的依赖，以此类推：

test/injector\_spec.js

```js
it('loads the transitively required modules of a module', function() {
  var module1 = window.angular.module('myModule', []);
  var module2 = window.angular.module('myOtherModule', ['myModule']);
  var module3 = window.angular.module('myThirdModule', ['myOtherModule']);
  module1.constant('aConstant', 42);
  module2.constant('anotherConstant', 43);
  module3.constant('aThirdConstant', 44);
  var injector = createInjector(['myThirdModule']);

  expect(injector.has('aConstant')).toBe(true);
  expect(injector.has('anotherConstant')).toBe(true);
  expect(injector.has('aThirdConstant')).toBe(true);
});
```

要让这两个特性生效很简单。当我们加载一个模块时，在执行任务队列之前，我们首先对该模块的依赖进行递归加载。当然，要递归的话，我们需要先对模块加载函数起一个名字 loadModule：

src/injector.js

```js
_.forEach(modulesToLoad, function loadModule(moduleName) {
  var module = window.angular.module(moduleName);
  _.forEach(module.requires, loadModule);
  _.forEach(module._invokeQueue, function(invokeArgs) {
    var method = invokeArgs[0];
    var args = invokeArgs[1];
    $provide[method].apply($provide, args);
  });
});
```

当我们实现模块依赖其他模块时，很可能会遇到循环依赖的问题：

test/injector\_spec.js

```js
it('loads each module only once', function() {
  window.angular.module('myModule', ['myOtherModule']);
  window.angular.module('myOtherModule', ['myModule']);

  createInjector(['myModule']);
});
```

当我们进行上面的测试时，会产生调用堆栈溢出的问题，因为两个模块相互依赖，传递依赖一直在两个模块之间发生。

解决循环依赖的方法是让每一个模块只会被加载一次。这也能避免多个模块依赖同一个模块时，被依赖模块会多次实例化的问题：

src/injector.js

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
    }
  };
}
```



