### 注册控制器（Controller Registration）

虽然我们有了一个好的开头，但目前这个“幼年”的`$controller`的 provider 并没有起到什么作用。确实。目前它里面只包裹了一个`$injector.instantiate`的调用而已。但如果考虑到 provider 的一个更典型的用例，这个情况就会发生改变。我们可以在配置阶段就注册控制器，而在运行时调用它们。

这里我们会在 provider 上使用一个新的方法，名为`register`，这个方法是用于在 config 代码块中注册控制器构造函数的。之后我们会通过名称来请求生成一个控制器的实例。就像之前一样，我们会期望获得控制器构造函数的一个实例：

_test/controller_spec.js_

```js
it('allows registering controllers at confg time', function() {
  function MyController() {}
  var injector = createInjector(['ng', function($controllerProvider) {
    $controllerProvider.register('MyController', MyController);
  }]);
  var $controller = injector.get('$controller');
  
  var controller = $controller('MyController');
  expect(controller).toBeDefned();
  expect(controller instanceof MyController).toBe(true);
});
```

`$controller` provider 会把已经注册的构造函数都存放到一个内部对象变量中，控制器名称就是 key，而构造函数本身就是 value。你可以使用 provider 里面的`register`方法直接添加注册构造函数：

_src/controller.js_

```js
function $ControllerProvider() {
  var controllers = {};

  this.register = function(name, controller) {
    controllers[name] = controller;
  };

  // this.$get = ['$injector', function($injector) {

  //   return function(ctrl, locals) {
  //     return $injector.instantiate(ctrl, locals);
  //   };

  // }];
}
```

现在，实际的`$controller`函数可以通过检查第一个参数的类型，决定是直接生成一个构造函数的实例，还是找到之前已经注册了的构造函数：

```js
this.$get = ['$injector', function($injector) {

  // return function(ctrl, locals) {
    if (_.isString(ctrl)) {
      ctrl = controllers[ctrl];
    }
  //   return $injector.instantiate(ctrl, locals);
  // };

}];
```

现在我们需要在`controller.js`文件中引进 LoDash：

```js
'use strict';

var _ = require('lodash');
```

跟指令类似，我们只调用一次`$controllerProvider.register`就可以注册几个控制器，只需要我们用一个对象来包裹我们需要注册的多个控制器，key 是控制器名称，而 value 是控制器的构造函数：

_test/controller_spec.js_

```js
it('allows registering several controllers in an object', function() {
  function MyController() {}
  function MyOtherController() {}
  var injector = createInjector(['ng', function($controllerProvider) {
    $controllerProvider.register({
      MyController: MyController,
      MyOtherController: MyOtherController
    });
  }]);
  var $controller = injector.get('$controller');

  var controller = $controller('MyController');
  var otherController = $controller('MyOtherController');

  expect(controller instanceof MyController).toBe(true);
  expect(otherController instanceof MyOtherController).toBe(true);
});
```

如果`register`传入的是一个对象，考虑到存储已注册构造函数的对象也是相同的结果，我们可以直接把传入的对象 extend 到这个对象中去：

_src/controller.js_

```js
this.register = function(name, controller) {
  if (_.isObject(name)) {
    _.extend(controllers, name);
  } else {
    // controllers[name] = controller;
  }
};
```

对于 Angular 应用开发者来说，`$controllerProvider`的`register`函数并不算是很熟悉。这是因为使用模块来注册控制器是一种更普遍的方式。模块对象有一个叫`controller`的方法，可以用于在模块上注册控制器：

_test/controller_spec.js_

```js
it('allows registering controllers through modules', function() {
  var module = window.angular.module('myModule', []);
  module.controller('MyController', function MyController() { });

  var injector = createInjector(['ng', 'myModule']);
  var $controller = injector.get('$controller');
  var controller = $controller('MyController');

  expect(controller).toBeDefned();
});
```

而我们实际在模块对象上加入的 controller 方法，只是简单地把我们刚才创建的`$controllerProvider.register`方法进行队列调用。当你调用`module.controller`之后，`$controllerProvider.register`会在模块加载后被调用：

_src/loader.js_

```js
var moduleInstance = {
  // name: name,
  // requires: requires,
  // constant: invokeLater('$provide', 'constant', 'unshift'),
  // provider: invokeLater('$provide', 'provider'),
  // factory: invokeLater('$provide', 'factory'),
  // value: invokeLater('$provide', 'value'),
  // service: invokeLater('$provide', 'service'),
  // decorator: invokeLater('$provide', 'decorator'),
  // flter: invokeLater('$flterProvider', 'register'),
  // directive: invokeLater('$compileProvider', 'directive'),
  controller: invokeLater('$controllerProvider', 'register'),
  // confg: invokeLater('$injector', 'invoke', 'push', confgBlocks),
  // run: function(fn) {
  //   moduleInstance._runBlocks.push(fn);
  //   return moduleInstance;
  // },
  // _invokeQueue: invokeQueue,
  // _confgBlocks: confgBlocks,
  // _runBlocks: []
};
```