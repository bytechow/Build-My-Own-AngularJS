### 链接之前编译过的指令（Linking Directives that Were Compiled Earlier）

在我们可以宣告异步编译功能已经开发完成之前，还有一些情况我们要考虑。首先，现在有一个明显的遗漏的情况，就是在同一个元素上，在遇到带异步模板的指令之前，对已经编译的指令进行链接的问题。我们之前将这些指令的链接函数都存放在`preLinkFns`和`postLinkFns`列表中，但当我们用延迟的节点链接函数取代这些链接函数，这些链接函数就都被丢弃了。

_test/compile_spec.js_

```js
it('links directives that were compiled earlier', function() {
  var linkSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        link: linkSpy
      };
    },
    myOtherDirective: function() {
      return {
        templateUrl: '/my_other_directive.html'
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    var linkFunction = $compile(el);
    
    $rootScope.$apply();
    
    linkFunction($rootScope);

    requests[0].respond(200, {}, '<div></div>');
    expect(linkSpy).toHaveBeenCalled();
    expect(linkSpy.calls.argsFor(0)[0]).toBe($rootScope);
    expect(linkSpy.calls.argsFor(0)[1][0]).toBe(el[0]);
    expect(linkSpy.calls.argsFor(0)[2].myDirective).toBeDefned();
  });
});
```

在这里，我们之前引入的`previousCompileContext`对象就能派上用场了。在两次调用`applyDirectivesToNode`之间需要把链接前（pre-link）函数和链接后（post-link）函数都进行缓存。我们需要在调用`compileTemplateUrl`时把这两个列表都放到上下文中：

_src/compile.js_

```js
nodeLinkFn = compileTemplateUrl(
  // _.drop(directives, i),
  // $compileNode,
  attrs, {
    // templateDirective: templateDirective,
    preLinkFns: preLinkFns,
    postLinkFns: postLinkFns
  }
);
```

当然，之后我们就需要在二次调用`applyDirectivesToNode`时对它们进行接收：

```js
function applyDirectivesToNode(
  directives, compileNode, attrs, previousCompileContext) {
  previousCompileContext = previousCompileContext || {};
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  var preLinkFns = previousCompileContext.preLinkFns || [];
  var postLinkFns = previousCompileContext.postLinkFns || [];
  var controllers = {};
  // var newScopeDirective, newIsolateScopeDirective;
  // var templateDirective = previousCompileContext.templateDirective;
  // var controllerDirectives;
  // ...
}
```

现在，我们在异步模板加载过程中始终保存着相同的链接函数列表。当我们调用这些链接函数时，我们对所有的链接函数都进行调用，不管他们是在模板加载之前构建还是之后。