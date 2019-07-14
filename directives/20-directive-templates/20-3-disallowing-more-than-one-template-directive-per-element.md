### 不允许元素使用超过一个指令模板（Disallowing More Than One Template Directive Per Element）

因为指令模板会替换元素原本的内容，那对一个元素应用几个带模板的指令就没有什么必要了，只有最后一个指令的模板会保留下来。

Angular 在你尝试这样做的时候会显式地抛出一个异常，因此如果发生这种问题，应用开发者可以很明确地知道：

_test/compile_spec.js_

```js
it('does not allow two directives with templates', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        template: '<div></div>'
      };
    },
    myOtherDirective: function() {
      return {
        template: '<div></div>'
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive my-other-directive></div>');
    expect(function() {
      $compile(el);
    }).toThrow();
  });
});
```

要完成这种提示，我们需要做的与检查重复的继承作用域相似。我们会引进一个变量来记录我们遇见的带模板指令：

_src/compile.js_

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // var preLinkFns = [],
  //   postLinkFns = [],
  //   controllers = {};
  // var newScopeDirective, newIsolateScopeDirective;
  var templateDirective;
  // var controllerDirectives;

  // ...
  
}
```

有了这个变量后，我们在遇到带模板指令时，就把它放到这个变量中，并且检查之前是否已经遇到了另一个带模板指令：

```js
if (directive.template) {
  if (templateDirective) {
    throw 'Multiple directives asking for template';
  }
  templateDirective = directive;
  // $compileNode.html(directive.template);
}
```