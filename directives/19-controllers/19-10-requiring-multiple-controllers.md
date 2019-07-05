### 请求引入多个控制器（Requiring Multiple Controllers）

实际上，我们还可以引入不止一个指令控制器到指令中。如果你对`require`的值指定一个字符数组，那么链接函数的第四个参数会变成一个对应的控制器对象数组：

_test/compile_spec.js_

```js
it('can be required from multiple sibling directives', function() {
  function MyController() {}

  function MyOtherController() {}
  var gotControllers;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: true,
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        scope: true,
        controller: MyOtherController
      };
    });
    $compileProvider.directive('myThirdDirective', function() {
      return {
        require: ['myDirective', 'myOtherDirective'],
        link: function(scope, element, attrs, controllers) {
          gotControllers = controllers;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive my-third-directive></div>');
    $compile(el)($rootScope);
    expect(gotControllers).toBeDefned();
    expect(gotControllers.length).toBe(2);
    expect(gotControllers[0] instanceof MyController).toBe(true);
    expect(gotControllers[1] instanceof MyOtherController).toBe(true);
  });
});
```

在这个测试用例中，我们对一个元素指定了三个指令。前两个定义了控制器，而最后一个引入前两个指令的控制器。然后我们只需要检查第三个指令的链接函数接收的是不是这两个控制器就可以了。

要让这个功能通过其实十分简单。如果传递给`getControllers`的`require`参数是一个数组，我们就会递归访问`require`数组，并返回一个控制器数组：

_src/compile.js_

```js
function getControllers(require) {
  if (_.isArray(require)) {
    return _.map(require, getControllers);
  } else {
    // var value;
    // if (controllers[require]) {
    //   value = controllers[require].instance;
    // }
    // if (!value) {
    //   throw 'Controller ' + require + ' required by directive, cannot be found!';
    // }
    // return value;
  }
}
```