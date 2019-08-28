### 在组件内引入控制器（Requiring from Components）

组件的最后一个特性`require`，也是已经在指令实现的特性。它用于获取同一个元素或上层元素的指令的控制器，并以此实现跨指令（或跨组件）通信。

_test/compile_spec.js_

```js
it('may require other directive controllers', function() {
  var secondControllerInstance;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.component('first', {
      controller: function() {}
    });
    $compileProvider.component('second', {
      require: { first: '^' },
      controller: function() {
        secondControllerInstance = this;
      }
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<first><second></second></first>');
    $compile(el)($rootScope);
    expect(secondControllerInstance.first).toBeDefined();
  });
});
```

开发起来也不是什么难事，因为指令的`require`特性也已经实现了。我们只需要将组件配置对象中的`require`的属性传递给指令进行处理就可以了：

_src/compile.js_

```js
function factory($injector) {
  return {
    // restrict: 'E',
    // controller: options.controller,
    // controllerAs: options.controllerAs ||
    //               identifierForController(options.controller) ||
    //               '$ctrl',
    // scope: {},
    // bindToController: options.bindings || {},
    // template: makeInjectable(options.template, $injector),
    // templateUrl: makeInjectable(options.templateUrl, $injector),
    // transclude: options.transclude,
    require: options.require
  }; 
}
```

这样我们就已经把所有组件定义的 API 都覆盖到了。我们现在已经知道组件是怎么运作的、怎么在指令系统的基础上构建起来的，还有它与指令的区别在哪。