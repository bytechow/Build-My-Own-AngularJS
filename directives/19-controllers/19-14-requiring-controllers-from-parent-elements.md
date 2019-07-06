### 注入父元素上的指令（Requiring Controllers from Parent Elements）

从兄弟指令上引入控制器是一种情况，但它实际上是非常局限的：我们现在跟元素树的其他指令进行协作（在作用域对象上共享数据以外的）。

`require`标识实际上我们之前看到的更灵活，它允许我们注入父元素上的指令控制器而不仅仅是当前元素上的。这种注入模式可以通过在要注入的指令名称前面加`^`来实现：

_test/compile_spec.js_

```js
it('can be required from a parent directive', function() {
  function MyController() {}
  var gotMyController;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: {},
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        require: '^myDirective',
        link: function(scope, element, attrs, myController) {
          gotMyController = myController;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive><div my-other-directive></div></div>');
    $compile(el)($rootScope);
    expect(gotMyController).toBeDefned();
    expect(gotMyController instanceof MyController).toBe(true);
  });
});
```