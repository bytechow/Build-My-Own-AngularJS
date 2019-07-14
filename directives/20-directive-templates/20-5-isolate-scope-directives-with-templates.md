### 带模板的独立作用域指令（Isolate Scope Directives with Templates）

在前面的章节中，当我们实现独立作用域时，我们介绍过一个独立作用域只会用于请求它的指令，而不会用于在同一元素上的其他指令或它的子元素。

其实还有一个例外，这个例外跟模板有关：当一个指令既定义了独立作用域，又定义了模板时，模板里面的指令也会接收到这个独立作用域（或独立作用域的一个继承者）。也就是说，模板内容会被视为独立作用域的一个部分。如果你把这种带模板的独立作用域指令视作一个“拥有”属于自己的模板的组件，会好理解一点。

_test/compile_spec.js_

```js
it('uses isolate scope for template contents', function() {
  var linkSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: {
          isoValue: '=myDirective'
        },
        template: '<div my-other-directive></div>'
      };
    },
    myOtherDirective: function() {
      return {
        link: linkSpy
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive="42"></div>');
    $compile(el)($rootScope);
    expect(linkSpy.calls.frst().args[0]).not.toBe($rootScope);
    expect(linkSpy.calls.frst().args[0].isoValue).toBe(42);
  });
});
```

在节点链接函数里，当我们调用子节点的链接函数时，我们目前使用的还是上文的作用域，而不存在独立作用域。现在我们要对此进行修改，这样的话，当我们遇到一个带模板的独立作用域指令时，我们会将这个独立作用域跟子节点进行链接。我们能这样做的依据是，如果这个元素如果存在子节点，那这些子节点必然是出自独立作用域指令的模板：

_src/compile.js_

```js
// _.forEach(preLinkFns, function(linkFn) {
//   linkFn(
//     linkFn.isolateScope ? isolateScope : scope,
//     $element,
//     attrs,
//     linkFn.require && getControllers(linkFn.require, $element)
//   );
// });
// if (childLinkFn) {
  var scopeToChild = scope;
  if (newIsolateScopeDirective && newIsolateScopeDirective.template) {
    scopeToChild = isolateScope;
  }
  childLinkFn(scopeToChild, linkNode.childNodes);
// }
// _.forEachRight(postLinkFns, function(linkFn) {
//   linkFn(
//     linkFn.isolateScope ? isolateScope : scope,
//     $element,
//     attrs,
//     linkFn.require && getControllers(linkFn.require, $element)
//   );
// });
```