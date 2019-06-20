### 查找全局控制器（Global Controller Lookup）

刚才那种注册控制器的方法是推荐在 Angular 应用中使用的。但`$controller`服务还提供另一种方法，可以从全局的`window`对象上查找控制器构造函数。然而，默认情况下是不允许这样做的，这时使用这种查找方法会抛出一个异常：

_test/controller_spec.js_

```js
it('does not normally look controllers up from window', function() {
  window.MyController = function MyController() {};
  var injector = createInjector(['ng']);
  var $controller = injector.get('$controller');
  
  expect(function() {
    $controller('MyController');
  }).toThrow();
});
```

相反，如果我们在配置应用期间使用`$controllerProvider`服务调用一个叫做`allowGlobals`的特殊函数，`$controller`就可以在`window`对象上查找构造函数并对其进行使用：

```js
it('looks up controllers from window when so confgured', function() {
  window.MyController = function MyController() {};
  var injector = createInjector(['ng', function($controllerProvider) {
    $controllerProvider.allowGlobals();
  }]);
  var $controller = injector.get('$controller');
  var controller = $controller('MyController');
  expect(controller).toBeDefned();
  expect(controller instanceof window.MyController).toBe(true);
});
```

实际上我们不推荐使用这种配置选项，因为这样不利于模块化。这种查找方法仅适用于一些示例应用的简化，就算是这样也会是一个“可疑”的操作。虽然如此，但既然已经存在，我们就来看一下它是怎么起作用的：

```js
function $ControllerProvider() {

  // var controllers = {};
  var globals = false;
  
  this.allowGlobals = function() {
    globals = true;
  };

  // this.register = function(name, controller) {
  //   if (_.isObject(name)) {
  //     _.extend(controllers, name);
  //   } else {
  //     controllers[name] = controller;
  //   }
  // };

  // this.$get = ['$injector', function($injector) {

  //   return function(ctrl, locals) {
  //     if (_.isString(ctrl)) {
        if (controllers.hasOwnProperty(ctrl)) {
          // ctrl = controllers[ctrl];
        } else if (globals) {
          ctrl = window[ctrl];
        }
      // }
  //     return $injector.instantiate(ctrl, locals);
  //   };

  // }];
  
}
```