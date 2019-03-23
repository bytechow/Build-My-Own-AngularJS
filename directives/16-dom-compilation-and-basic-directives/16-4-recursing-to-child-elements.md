### 递归处理子元素节点（Recursing to Child Elements）

目前我们实现的简单指令只能遍历顶层元素。我们也应该允许指令编译顶元素的子节点：

_test/compile_spec.js_

```js
it('compiles element directives from child elements', function() {
  var idx = 1;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', idx++);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div><my-directive></my-directive></div>');
    $compile(el);
    expect(el.data('hasCompiled')).toBeUndefined();
    expect(el.find('> my-directive').data('hasCompiled')).toBe(1);
  });
});
```

这里我们会检查子元素`<my-directive>`是否会被编译。作为一个健全的测试，我们也需要检查一些父元素`<div>`是否不会编译。

我们的编译器也应该可以支持编译嵌套的指令元素：

```js
it('compiles nested directives', function() {
  var idx = 1;
  var injector = makeInjectorWithDirectives('myDir', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', idx++);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<my-dir><my-dir><my-dir></my-dir></my-dir></my-dir>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(1);
    expect(el.fnd('> my-dir').data('hasCompiled')).toBe(2);
    expect(el.fnd('> my-dir > my-dir').data('hasCompiled')).toBe(3);
  });
});
```

要满足这些需求其实非常简单。我们需要做的就是检测当前节点有没有子节点，若有，则以子节点集合作为参数继续调用`compileNodes`:

_src/compile.js_

```js
function compileNodes($compileNodes) {
  _.forEach($compileNodes, function(node) {
    var directives = collectDirectives(node);
    applyDirectivesToNode(directives, node);
    if (node.childNodes && node.childNodes.length) {
      compileNodes(node.childNodes);
    }
  });
}
```

要注意编译的执行顺序是非常重要的：我们会先编译父节点，再编译子节点。