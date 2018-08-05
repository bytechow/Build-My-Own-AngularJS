### Config 服务（Config Blocks）

本章要介绍的是配置服务，这是我们采用 provider 而不是高阶服务 factory 或者 service 的原因之一。从之前的实现，我们可以知道在某个 provider 的 $get 方法调用之前，你可以对这个 provider 进行配置，也就是说我们通过这种手段可以影响依赖的实例化过程。Angular的路由组件——ngRoute中的$route服务就是一个例子，$route 服务是用于将 URL 映射到指定的控制器，也就是我们通常所说的“路由”，要配置路由就得使用 $route 的 provider，也即是 $routeProvider

```js
$routeProvider.when('/someUrl', {
  templateUrl: '/my/view.html',
  controller: 'MyController'
})
```

要使用 $routeProvider，我们需要 provider 注射器，但以目前的代码实现来看，要完成这项工作，还只能在 provider 构造函数中做到。为了配置而定义一个 provider 函数并不方便。我们需要的是一个可以在模块加载期间进行任意配置的方法，另外也要实现对这个方法注入 provider。这个方法在 Angular 中就是 config 服务了。

我们可以通过模块实例的 config 方法增加一个配置服务。这个配置服务的值是一个函数，当我们的 injector 被创建时，这个函数就会被调用：

test/injector_spec.js

```js
it('runs confg blocks when the injector is created', function() {
  var module = window.angular.module('myModule', []);
  var hasRun = false;
  module.confg(function() {
    hasRun = true;
  });
  createInjector(['myModule']);
  expect(hasRun).toBe(true);
});
```

当然，配置服务函数是允许注入依赖的，注入的方法依然可以是我们之前实现的三种方式之一。比如，你可以注入 $provider：

test/injector_spec.js

```js
it('injects confg blocks with provider injector', function() {
  var module = window.angular.module('myModule', []);
  module.confg(function($provide) {
    $provide.constant('a', 42);
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(42);
});
```

所以，我们可以说 config 服务就是一个允许注入 provider 依赖的函数。我们可以通过 providerInjector.invoke 来调用配置服务，来完成这一需求：

首先，我们需要在模块实例中新增配置服务的接口。我们需要对要调用 providerInjector.invoke 方法的任务进行重新排序。要进行调整的话，我们首先会遇到的问题是，目前我们所以的任务都是基于 $provider 对象的方法执行，而没有用到 $injector。我们将对此进行改进，我们将会对任务队列创建方法（也就是 invokeLater ）进行改造，它的参数将会改变为：1）任务执行方法所在的对象， 2) 任务执行方法名称 3) 以那种方法插入任务

src/loader.js

```js
var invokeLater = function(service, method, arrayMethod) {
  return function() {
    var item = [service, method, arguments];
    invokeQueue[arrayMethod || 'push'](item);
    return moduleInstance;
  };
};
```

然后，我们需要更新现有的任务队列创建方法，明确地把 $provider 指定为 constant 和 provider 任务的任务执行方法所在的对象：

src/loader.js

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: invokeLater('$provide', 'constant', 'unshift'),
  provider: invokeLater('$provide', 'provider'),
  _invokeQueue: invokeQueue
};
```

另一个问题是，目前按照我们的代码实现，我们所有的任务都在同一个队列中，调用顺序跟注册顺序是一致的。但当我们要加入 config 服务，我们就要进行改变了。简单来说，我们希望所有的注册工作可以在 config 服务之前完成。这样我们才能保证，当 config 服务函数被执行时，所有的 provider 已经处于可用状态，不管他们是在 config 服务之前还是之后注册的：

test/injector_spec.js

```js
it('allows registering confg blocks before providers', function() {
  var module = window.angular.module('myModule', []);
  module.confg(function(aProvider) {});
  module.provider('a', function() {
    this.$get = _.constant(42);
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(42);
});
```

要实现不依赖注册顺序，我们需要单独为 config 任务建立一个任务队列。具体上，我们会对 invokeLater 增加最后一个参数 queue，默认指向 invokeQueue：

src/loader.js

```js
var createModule = function(name, requires, modules, confgFn) {
  if (name === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid module name';
  }
  var invokeQueue = [];
  var confgBlocks = [];
  var invokeLater = function(service, method, arrayMethod, queue) {
    return function() {
      queue = queue || invokeQueue;
      queue[arrayMethod || 'push']([service, method, arguments]);
      return moduleInstance;
    };
  };
  // ...
}
```

现在，我们得新增一个任务队列方法，这个队列中的任务会通过 $injector.invoke 方法进行执行。另外，还需要在模块实例中新增 config 服务任务队列 configBlock，这样我们才可以在 injector 里面逐个执行 config 任务。

src/loader.js

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: invokeLater('$provide', 'constant', 'unshift'),
  provider: invokeLater('$provide', 'provider'),
  confg: invokeLater('$injector', 'invoke', 'push', confgBlocks),
  _invokeQueue: invokeQueue,
  _confgBlocks: confgBlocks
};
```

在 injector 中，我们需要遍历两个任务队列，因此我们可以抽离出一个专门用于遍历执行任务队列的方法 runInvokeQueue：

src/loader.js

```js
function runInvokeQueue(queue) {
  _.forEach(queue, function(invokeArgs) {
    var method = invokeArgs[0];
    var args = invokeArgs[1];
    providerCache.$provide[method].apply(providerCache.$provide, args);
  });
}
_.forEach(modulesToLoad, function loadModule(moduleName) {
  if (!loadedModules.hasOwnProperty(moduleName)) {
    loadedModules[moduleName] = true;
    var module = window.angular.module(moduleName);
    _.forEach(module.requires, loadModule);
    runInvokeQueue(module._invokeQueue);
    runInvokeQueue(module._confgBlocks);
  }
}) 
```

由于任务执行的服务对象不一定是 $provider，我们会在每一个任务注册时新增一个 service 参数，且作为第一个参数，我们需要进行修改：

src/loader.js

```js
function runInvokeQueue(queue) {
  _.forEach(queue, function(invokeArgs) {
    var service = providerInjector.get(invokeArgs[0]);
    var method = invokeArgs[1];
    var args = invokeArgs[2];
    service[method].apply(service, args);
  });
}
```

现在，我们可以通过调用模块实例的 config 方法来增加一个配置任务。我们还会提供另一种简便的方式，就是对 angular.module 新增一个可选参数：

test/injector_spec.js

```js
it('runs a confg block added during module registration', function() {
  var module = window.angular.module('myModule', [], function($provide) {
    $provide.constant('a', 42);
  });
  var injector = createInjector(['myModule']);
  expect(injector.get('a')).toBe(42);
});
```

如何有这么一个函数作为模块注册的参数之一，我们需要先把它传递到 createModule 方法中去：

src/loader.js

```js
ensure(angular, 'module', function() {
  var modules = {};
  return function(name, requires, confgFn) {
    if (requires) {
      return createModule(name, requires, modules, confgFn);
    } else {
      return getModule(name, modules);
    }
  };
});
```

而在 createModule 内部，我们可以直接调用模块实例的 config 方法进行注册：

src/loader.js

```js
var createModule = function(name, requires, modules, confgFn) {
  if (name === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid module name';
  }
  var invokeQueue = [];
  var confgBlocks = [];
  var invokeLater = function(service, method, arrayMethod, queue) {
    return function() {
      queue = queue || invokeQueue;
      queue[arrayMethod || 'push']([service, method, arguments]);
      return moduleInstance;
    };
  };
  var moduleInstance = {
    name: name,
    requires: requires,
    constant: invokeLater('$provide', 'constant', 'unshift'),
    provider: invokeLater('$provide', 'provider'),
    confg: invokeLater('$injector', 'invoke', 'push', confgBlocks),
    _invokeQueue: invokeQueue,
    _confgBlocks: confgBlocks
  };
  if (confgFn) {
    moduleInstance.confg(confgFn);
  }
  modules[name] = moduleInstance;
  return moduleInstance;
};
```
