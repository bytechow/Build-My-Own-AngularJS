### 注册常量（Registering A Constant）

我们要接触的第一个 Angular 应用组件是常量（constants）。通过常量组件，我们可以在模块中注册一些简单值，比如数字、字符串、对象、函数等。

当我们在一个模块中注册了一个常量，并以该模块作为参数生成注射器，我们就可以使用 injector 的 has 方法来检查常量是否成功注册到注射器中：

test/injector\_spec.js

```js
it('has a constant that has been registered to a module', function() {
  var module = window.angular.module('myModule', []);
  module.constant('aConstant', 42);
  var injector = createInjector(['myModule']);
  expect(injector.has('aConstant')).toBe(true);
});
```

现在我们终于看到了从定义模块到创注射器的一个完整过程。我们可以发现一个有趣的现象，代码中传递给 createInjector 的是模块的名称，而不是模块实例的引用，最终我们会在 angular.module 中根据模块名称查找对应的模块实例。

为了让检测更严谨，我们希望当调用 has 方法无法找到模块时会返回 false：

test/injector\_spec.js

```js
it('does not have a non-registered constant', function() {
  var module = window.angular.module('myModule', []);
  var injector = createInjector(['myModule']);
  expect(injector.has('aConstant')).toBe(false);
});
```

那么问题来了，我们应该怎么做才能让注册在模块中的常量能在注射器中使用？首先，我们需要一个在模块实例中新增一个注册方法 constant：

src/loader.js

```js
var createModule = function(name, requires, modules) {
  if (name === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid module name';
  }
  var moduleInstance = {
    name: name,
    requires: requires,
    constant: function(key, value) {}
  };
  modules[name] = moduleInstance;
  return moduleInstance;
};
```

Angular有一条通用的规则——模块实际上不持有任何应用组件。他们只保存了创建应用组件的"食谱"（recipes），应用组件实际上会被保存在注射器中。

换句话说，模块实例中包含一系列任务，如“注册一个常量”，这些任务最终会在注射器加载模块时被执行。这一系统任务在 Angular 中被称作“任务调用队列”（invoke queue）。所有模块都会有一个任务调用队列。

目前，我们会将任务调用队列定义为一个二维数组。即在队列中的每个元素也是一个数组，这个数组包含两个元素：应用组件的类型和注册组件所需的参数。就拿上面的单元测试作为例子，我们要注册一个值为42的常量，其任务队列如下：

```js
[
  ['constant', ['aConstant', 42]]
]
```

任务队列将会保存在模块实例的 \_invokeQueue（有下划线前缀代表该方法被认为是私有的）方法。我们现在会在 createModule 中新增一个私有数组变量 invokeQueue ，当我们调用 constant 方法时，我们会向 invokeQueue 新增一个任务：

src/loader.js

```js
var createModule = function(name, requires, modules) {
  if (name === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid module name';
  }
  var invokeQueue = [];
  var moduleInstance = {
    name: name,
    requires: requires,
    constant: function(key, value) {
      invokeQueue.push(['constant', [key, value]]);
    },
    _invokeQueue: invokeQueue
  };
  modules[name] = moduleInstance;
  return moduleInstance;
};
```

当我们创建注射器后，我们应该遍历（数组参数中）所有的模块名称，找到对应的模块实例后，再依次执行模块实例中的任务队列 \_invokeQueue：

src/injector.js

```js
'use strict';
var _ = require('lodash');

function createInjector(modulesToLoad) {
  _.forEach(modulesToLoad, function(moduleName) {
    var module = window.angular.module(moduleName);
    _.forEach(module._invokeQueue, function(invokeArgs) {});
  });
  return {};
}
module.exports = createInjector;
```

在注射器中，我们提供了一些内部方法处理任务队列中任务。我们将会新建一个 $provide 对象（我们之后会解释为什么叫这个名字）。当我们遍历任务队列，我们先找到第一个任务参数，这个参数就是 $provide 对象中其中一个方法的名称（例如“constant“）。查找到了对应的方法后，我们会把第二个任务参数（也是一个数组）作为参数，对该方法进行调用：

src/injector.js

```js
function createInjector(modulesToLoad) {
  var $provide = {
    constant: function(key, value) {

    }
  };
  _.forEach(modulesToLoad, function(moduleName) {
    var module = window.angular.module(moduleName);
    _.forEach(module._invokeQueue, function(invokeArgs) {
      var method = invokeArgs[0];
      var args = invokeArgs[1];
      $provide[method].apply($provide, args);
    });
  });
  return {};
}
```

所以，当我们注册某个组件时（例如constant），实际上也就确定了在之后注射器实例化时会调用 $provide 中的哪个方法。当然，注册和调用不是同步执行，只有当模块被注射器加载 $provide 中的方法才会被调用。同时，这也解释了为什么我们需要把调用信息放到任务队列中去。

接下来要做的就是注册一个常量的逻辑。不同的应用组件需要不同的初始化逻辑，但它们都有一个共同点—— 一旦被创建，就会被缓存起来。缓存常量非常简单，所以我们从这里入手。接着，我们就可以实现注射器实例的 has 方法，这个方法可以通过 key 值从缓存中查找对应依赖：

src/injector.js

```js
function createInjector(modulesToLoad) {
  var cache = {};
  var $provide = {
    constant: function(key, value) {
      cache[key] = value;
    }
  };
  _.forEach(modulesToLoad, function(moduleName) {
    var module = window.angular.module(moduleName);
    _.forEach(module._invokeQueue, function(invokeArgs) {
      var method = invokeArgs[0];
      var args = invokeArgs[1];
      $provide[method].apply($provide, args);
    });
  });
  return {
    has: function(key) {
      return cache.hasOwnProperty(key);
    }
  };
}
```

由于我们在 has 方法里面也用到了 hasOwnProperty 方法检验是否存在对应的缓存，我们需要防止用户使用 hasOwnProperty 作为常量组件的名称：

src/injector\_spec.js

```js
it('does not allow a constant called hasOwnProperty', function() {
  var module = window.angular.module('myModule', []);
  module.constant('hasOwnProperty', false);
  expect(function() {
    createInjector(['myModule']);
  }).toThrow();
});
```

我们可以在 $provide 对象的 constant 方法对这种行为进行限制：

src/injector.js

```js
constant: function(key, value) {
  if (key === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid constant name!';
  }
  cache[key] = value;
}
```

除了提供 has 方法用于检查应用组件是否存在，注射器还提供了一个方法 get 用于获取应用组件：

test/injector\_spec.js

```js
it('can return a registered constant', function() {
  var module = window.angular.module('myModule', []);
  module.constant('aConstant', 42);
  var injector = createInjector(['myModule']);
  expect(injector.get('aConstant')).toBe(42);
});
```

暂时，我们就直接从缓存中获取对应的值就好：

src/injector.js

```js
return {
  has: function(key) {
    return cache.hasOwnProperty(key);
  },
  get: function(key) {
    return cache[key];
  }
};
```

> 大部分 Angular 依赖注入特性都需要模块加载器和注射器共同协作，模块加载器存在于 loader.js，注射器存在于 injector.js。我们会把大部分依赖注入相关的测试放到 injector\_spec.js，而 loader\_spec.js 只处理模块加载器相关的功能检验。



