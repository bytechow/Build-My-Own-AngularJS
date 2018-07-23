### 最简单的provider——一个包含$get方法的对象（The Simplest Possible Provider: An Object with A $get Method）

一般来说，在 AngularJS 中一个拥有 $get 方法的对象就可以称为 provider。当你把这个对象注册成为应用组件后，注射器就可以调用它的 $get 方法，调用方法的返回值就成为最终的依赖值。

test/injector\_spec.js

```js
it('allows registering a provider and uses its $get', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', {
    $get: function() {
      return 42;
    }
  });
  var injector = createInjector(['myModule']);
  expect(injector.has('a')).toBe(true);
  expect(injector.get('a')).toBe(42);
});
```

在这个测试用例中，{ $get: function\( \) { return 42; } } 这个对象就是依赖 a 的 provider。虽然现在你会觉得 provider 生成依赖比较啰嗦，但很快你就会发现，provider 让我们有机会在依赖 a 生成之前做一些计算或配置，这是 constant 服务无法比拟的。

要实现 provider 方法，我们要从模块加载器开始。首先要在模块实例中加入 provider 方法，该方法可以新增一个注册 provider 的任务到任务队列中，就像之前 constant 一样。

src/loader.js

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: function(key, value) {
    invokeQueue.push(['constant', [key, value]]);
  },
  provider: function(key, provider) {
    invokeQueue.push(['provider', [key, provider]]);
  },
  _invokeQueue: invokeQueue
};
```

这里的 constant 和 provider 方法大同小异，我们可以抽离出一个公共方法：

src/loader.js

```js
var createModule = function(name, requires, modules) {
  if (name === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid module name';
  }
  var invokeQueue = [];
  var invokeLater = function(method) {
    return function() {
      invokeQueue.push([method, arguments]);
      return moduleInstance;
    };
  };
  var moduleInstance = {
    name: name,
    requires: requires,
    constant: invokeLater('constant'),
    provider: invokeLater('provider'),
    _invokeQueue: invokeQueue
  };
  modules[name] = moduleInstance;
  return moduleInstance;
};
```

调用 invokeLater 后会返回一个函数，该函数将实际执行将某类任务（目前只有 constant 和 provider 两类）插入到任务队列的逻辑。这里的关键是，具体执行哪一类方法是由调用 invokeLater 时传入的参数决定，继而决定了生成依赖的方法（注射器内部对象变量 $provider 的方法） 。

> 上面这种对函数进行“预配置”的技术叫[柯里化](https://en.wikipedia.org/wiki/Currying)

注意，为了实现链式注册，我们在每一次注册应用组件之后都要返回模块实例。链式注册如：

```js
module.constant('a', 42).constant('b', 43)
```

当然，我们还需要在注册器中加入 provider 生成依赖的方法。现在，为了通过测试，我们先把直接调用 $get 获取到的返回值作为依赖值缓存起来。

src/injector.js

```js
var $provide = {
  constant: function(key, value) {
    if (key === 'hasOwnProperty') {
      throw 'hasOwnProperty is not a valid constant name!';
    }
    cache[key] = value;
  },
  provider: function(key, provider) {
    cache[key] = provider.$get();
  }
};
```



