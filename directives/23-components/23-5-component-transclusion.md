### 组件 transclusion（Component Transclusion）

Transclusion 是指将一个 DOM 片段传递给指令，让指令可以在内部的某个指定位置把它渲染出来的行为。组件也是完全支持这个特性的，实际上这个特性也是构建基于组件的应用是要用到的一个重要特性。

组件 transclusion 实现起来也并没有什么让人意外的。我们会使用`transclude`属性来启用组件的 transclusion 功能，然后我们就可以使用了。最简单的使用方法就是在指令模板的某个元素上加上`ng-transclude`指令。

_test/compile_spec.js_

```js
it('may use transclusion', function() {
  var injector = makeInjectorWithComponent('myComponent', {
    transclude: true,
    template: '<div ng-transclude></div>'
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component>Transclude me</my-component>');
    $compile(el)($rootScope);
    expect(el.find('div').text()).toEqual('Transclude me');
  }); 
});
```