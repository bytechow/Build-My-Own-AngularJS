### 获取已注册的模块（Getting A Registered Module）

angular.module 的另一个能力是可以获取一个已注册的模块。你只需要在调用时忽略第二个参数就好了：

test/loader\_spec.js

```js
it('allows getting a module', function() {
  var myModule = window.angular.module('myModule', []);
  var gotModule = window.angular.module('myModule');
  expect(gotModule).toBeDefined();
  expect(gotModule).toBe(myModule);
});
```

我们会使用另一个内部函数 getModule 来完成对模块的获取。另外，我们会利用闭包，通过内部变量 modules 来存储和获取已注册的模块。在代码中，无论是调用 createModule 进行创建，还是调用 getModule 进行获取，都把 modules 作为参数传入：

src/loader.js

```js
ensure(angular, 'module', function() {
  var modules = {};
  return function(name, requires) {
    if (requires) {
      return createModule(name, requires, modules);
    } else {
      return getModule(name, modules);
    }
  };
});
```

当然，现在我们在 createModule 方法中需要将新创建的模块实例保存到 modules 中：

src/loader.js

```js
var createModule = function(name, requires, modules) {
  var moduleInstance = {
    name: name,
    requires: requires
  };
  modules[name] = moduleInstance;
  return moduleInstance;
};
```

然后在 getModule 方法中，我们只需要从modules中读取即可：

src/loader.js

```js
var getModule = function(name, modules) {
  return modules[name];
};
```

从上面的代码我们了解到所有的模块都存储到内部变量 modules 中，这也是为什么我们需要保证 angular 全局对象和 angular.module 方法只会初始化一次，否则在之前注册的模块都会被清除掉。

当前，我们如果访问一个尚未注册的模块，结果会是 undefined。但 Angular 需要抛出一个异常来表明发生了异常，并说明异常原因：

test/loader\_spec.js

```js
it('throws when trying to get a nonexistent module', function() {
  expect(function() {
    window.angular.module('myModule');
  }).toThrow();
});
```

当我们使用 getModule 获取模块时，需要先检查该模块是否已经存在：

_src/loader.js_

```js
var getModule = function(name, modules) {
  if (modules.hasOwnProperty(name)) {
    return modules[name];
  } else {
    throw 'Module ' + name + ' is not available!';
  }
};
```

由于我们使用了 hasOwnProperty 方法来检查模块是否存在，我们要确保hasOwnProperty 不会被作为模块名称被注册，否则会覆盖 hasOwnProperty 方法，导致失效或者错误：

_src/loader\_spec.js_

```js
it('does not allow a module to be called hasOwnProperty', function() {
  expect(function() {
    window.angular.module('hasOwnProperty', []);
  }).toThrow();
});
```

我们在 createModule 函数中对这一行为进行拦截：

src/loader.js

```js
var createModule = function(name, requires, modules) {
  if (name === 'hasOwnProperty') {
    throw 'hasOwnProperty is not a valid module name';
  }
  var moduleInstance = {
    name: name,
    requires: requires
  };
  modules[name] = moduleInstance;
  return moduleInstance;
};
```













