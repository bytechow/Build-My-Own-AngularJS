### 注入跨元素指令的控制器（Requiring Controllers in Multi-Element Directives）

注入控制器的操作应该对跨元素指令也同样适用：

_test/compile_spec.js_

```js
it('is passed through grouped link wrapper', function() {
  function MyController() {}
  var gotMyController;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        multiElement: true,
        scope: {},
        controller: MyController,
        link: function(scope, element, attrs, myController) {
          gotMyController = myController;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive-start></div><div my-directive-end></div>');
    $compile(el)($rootScope);
    expect(gotMyController).toBeDefned();
    expect(gotMyController instanceof MyController).toBe(true);
  });
});
```

> 在这个测试用例中，我们使用了上一节实现的“自注入”特性：链接函数的第四个参数会是指令控制器。我们这样做只是为了简化测试用例，因为这样我们就不需要再定义其他指令了。

这个测试用例并没有立即通过的原因在于`groupElementsLinkFnWrapper`函数，这个函数是用于对跨元素指令的链接函数进行打包的，目前它并没有接收到链接函数的第四个参数，也就无法对它进行传递了。要解决这个问题也很简单：

```js
function groupElementsLinkFnWrapper(linkFn, attrStart, attrEnd) {
  return function(scope, element, attrs, ctrl) {
    // var group = groupScan(element[0], attrStart, attrEnd);
    return linkFn(scope, group, attrs, ctrl);
  };
}
```