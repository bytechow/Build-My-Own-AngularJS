### run 服务（Run Blocks）

我们可以认为 run 服务是 config 服务的近亲，也是在注射器的构建阶段调用的函数：

test/injector\_spec.js

```js
it('runs run blocks when the injector is created', function() {
  var module = window.angular.module('myModule', []);
  var hasRun = false;
  module.run(function() {
    hasRun = true;
  });
  createInjector(['myModule']);
  expect(hasRun).toBe(true);
});
```

它们的区别在于，run 服务是从 instance 缓存里面注入依赖：

test/injector\_spec.js

```js
it('injects run blocks with the instance injector', function() {
  var module = window.angular.module('myModule', []);

  module.provider('a', {$get: _.constant(42)});

  var gotA;
  module.run(function(a) {
     gotA = a;
  });

  createInjector(['myModule']);

  expect(gotA).toBe(42);
});
```

显然，run服务无法配置 provider，但是可以允许我们在 Angular 应用启动时执行一些代码。要实现这一服务，我们只需要在模块加载器中提供接口，把要执行的代码收集起来，等注射器创建好了以后再进行调用即可：

src/loader.js

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: invokeLater('$provide', 'constant', 'unshift'),
  provider: invokeLater('$provide', 'provider'),
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

我们先尝试用最简单的方式进行实现。我可以把 run 服务任务队列放在 config 服务任务队列之后进行遍历执行即可：

src/injector.js

```js
_.forEach(modulesToLoad, function loadModule(moduleName) {
  if (!loadedModules.hasOwnProperty(moduleName)) {
    loadedModules[moduleName] = true;
    var module = window.angular.module(moduleName);
    _.forEach(module.requires, loadModule);
    runInvokeQueue(module._invokeQueue);
    runInvokeQueue(module._configBlocks);
    _.forEach(module._runBlocks, function(runBlock) {
      instanceInjector.invoke(runBlock);
    });
  }
});
```

目前这样能使测试用例通过，但还有一个问题。run 服务应该在模块加载完毕后被执行。当你的注射器要加载几个模块，他们的 run 服务应该等到所有模块都已经加载完才执行。但当前的实现并不满足：

test/injector\_spec.js

```js
it('configures all modules before running any run blocks', function() {
  var module1 = window.angular.module('myModule', []);
  module1.provider('a', {
    $get: _.constant(1)
  });
  var result;
  module1.run(function(a, b) {
    result = a + b;
  });
  var module2 = window.angular.module('myOtherModule', []);
  module2.provider('b', {
    $get: _.constant(2)
  });
  createInjector(['myModule', 'myOtherModule']);
  expect(result).toBe(3);
});
```

要解决这个问题也很简单，我们只需要用一个数组把 run 服务任务都收集起来，然后在所有模块都加载完之后，再从数组中取出 run 服务逐个执行：

src/injector.js

```js
var runBlocks = [];
_.forEach(modulesToLoad, function loadModule(moduleName) {
  if (!loadedModules.hasOwnProperty(moduleName)) {
    loadedModules[moduleName] = true;
    var module = window.angular.module(moduleName);
    _.forEach(module.requires, loadModule);
    runInvokeQueue(module._invokeQueue);
    runInvokeQueue(module._configBlocks);
    runBlocks = runBlocks.concat(module._runBlocks);
  }
});
_.forEach(runBlocks, function(runBlock) {
  instanceInjector.invoke(runBlock);
});
```

总结一下，config 服务会在模块加载期间执行，而 run 服务则会在模块加载后马上执行。

