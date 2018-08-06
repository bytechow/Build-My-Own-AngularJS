### run 服务（Run Blocks）

我们可以认为 run 服务是 config 服务的近亲，也是在注射器的构建阶段调用的函数：

test/injector_spec.js

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

test/injector_spec.js

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

明显，run服务无法配置 provider，但是可以允许我们在 Angular 应用启动时执行一些代码。要实现这一服务，我们只需要在模块加载器中提供接口，把要执行的代码收集起来，等注射器创建好了以后再进行调用即可：

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