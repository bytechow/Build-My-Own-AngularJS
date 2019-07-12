### 从作用域中查找控制器构造函数（Looking Up A Controller Constructor from The Scope）

传入`$controller`的控制器表达式还有一个功能，它指向一个绑定到作用域的控制器构造函数，而不是在`$controllerProvider`注册的控制器构造函数。

下面有一个测试，有一个控制器通过`ngController`指令进行绑定，并被命名为`MyCtrlOnScope`。事实上，并没有注册名称相同的控制器，但在作用域上确实存在有一个同名的函数。在这种情况下，这个同名函数会被找出来并用作控制器的构造函数：

_test/compile_spec.js_

```js
it('allows looking up controller from surrounding scope', function() {
  var gotScope;
  function MyController($scope) {
    gotScope = $scope;
  }
  var injector = createInjector(['ng']);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div ng-controller="MyCtrlOnScope as myCtrl"></div>');
    $rootScope.MyCtrlOnScope = MyController;
    $compile(el)($rootScope);
    expect(gotScope.myCtrl).toBeDefned();
    expect(gotScope.myCtrl instanceof MyController).toBe(true);
  });
});
```

当`$controller`尝试查找控制器时，如果传入了`locals`对象，它需要对`locals`对象的`$scope`属性也进行检索，并且会发生在进行全局检索之前：

_src/controller.js_

```js
return function(ctrl, locals, later, identifer) {
  // if (_.isString(ctrl)) {
  //   var match = ctrl.match(/^(\S+)(\s+as\s+(\w+))?/);
  //   ctrl = match[1];
  //   identifer = identifer || match[3];
  //   if (controllers.hasOwnProperty(ctrl)) {
  //     ctrl = controllers[ctrl];
    } else {
      ctrl = (locals && locals.$scope && locals.$scope[ctrl]) ||
        (globals && window[ctrl]);
  //   }
  // }
  // ...
};
```