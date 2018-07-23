### 注册模块（Regisering A Module）

做好了基础工作，现在我们终于可以进入正题——注册模块。

本章剩下的测试用例都要用到模块加载器（module loader），所以，我们会新建一个 describe 单元，在这个单元里使用 beforeEach 提前为每个用例初始化模块加载器：

test/loader\_spec.js

```js
describe('modules', function() {
  beforeEach(function() {
    setupModuleLoader(window);
  });
});
```

首先测试一下我们是否能通过调用 angular.module 函数获取一个模块对象

angular.module 接收两个参数，第一个参数是模块名称，第二个是一个包含模块依赖的数组（字符串数组，可能为空）。这个方法会返回一个模块对象，对象中的 name 属性对应模块名称：

test/loader\_spec.js

```js
it('allows registering a module', function() {
  var myModule = window.angular.module('myModule', []);
  expect(myModule).toBeDefined();
  expect(myModule.name).toEqual('myModule');
});
```

当我们用同一个名称重复注册模块，最后注册的模块将会覆盖之前注册的。这也意味着当我们用同一个名称作为参数调用 module 方法两次，我们将会得到两个不同的模块对象：

test/loader\_spec.js

```js
it('replaces a module when registered with same name again', function() {
  var myModule = window.angular.module('myModule', []);
  var myNewModule = window.angular.module('myModule', []);
  expect(myNewModule).not.toBe(myModule);
});
```

在 module 方法内部，我们将会把新建模块的工作交给 createModule 内部函数。当前，我们只需要简单地创建一个对象，然后返回它就可以了：

src/loader.js

```js
function setupModuleLoader(window) {
  var ensure = function(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
  };
  var angular = ensure(window, 'angular', Object);
  var createModule = function(name, requires) {
    var moduleInstance = {
      name: name
    };
    return moduleInstance;
  };
  ensure(angular, 'module', function() {
    return function(name, requires) {
      return createModule(name, requires);
    };
  });
}
```

模块实例除了能获取模块名称，也要能获取代表模块依赖的数组（requires）：

test/loader\_spec.js

```js
it('attaches the requires array to the registered module', function() {
  var myModule = window.angular.module('myModule', ['myOtherModule']);
  expect(myModule.requires).toEqual(['myOtherModule']);
});
```

我们只需要直接把 module 方法的第二个参数直接赋值为 module 实例的属性即可满足要求：

src/loader.js

```js
var createModule = function(name, requires) {
  var moduleInstance = {
    name: name,
    requires: requires
  };
  return moduleInstance;
};
```



