### 模板函数（Template Functions）

`template`属性的值并不一定只能是字符串。它也可以是一个返回值为字符串的函数。这个函数被调用时会传入两个参数：使用指令的元素 DOM 节点，还有元素节点的`Attributes`对象。这让我们有机会去动态构建模板：

_test/compile_spec.js_

```js
it('supports functions as template values', function() {
  var templateSpy = jasmine.createSpy()
    .and.returnValue('<div class="from-template"></div>');
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        template: templateSpy
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive></div>');
    $compile(el);
    expect(el.fnd('> .from-template').length).toBe(1);
    // Check that template function was called with element and attrs
    expect(templateSpy.calls.frst().args[0][0]).toBe(el[0]);
    expect(templateSpy.calls.frst().args[1].myDirective).toBeDefned();
  });
});
```

如果 template 属性值是一个函数，那我们会调用它来生成模板，而不是像字符串一样直接使用：

_src/compile.js_

```js
if (directive.template) {
  // if (templateDirective) {
  //   throw 'Multiple directives asking for template';
  // }
  // templateDirective = directive;
  $compileNode.html(_.isFunction(directive.template) ?
    directive.template($compileNode, attrs) :
    directive.template);
}
```