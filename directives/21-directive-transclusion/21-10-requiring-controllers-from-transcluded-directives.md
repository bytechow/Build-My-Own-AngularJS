### 从进行了 transclude 的指令引入控制器（Requiring Controllers from Transcluded Directives）

在关于控制器的章节中，我们实现了`require`配置项，这个配置项允许我们访问当前元素或祖先元素上的其他指令的控制器。transclusion 需要把元素内容从一处迁移到另一处，也就是需要更改 DOM 的祖先结构，那怎么跟这个配置项进行结合呢？

对于普通的 transclusion 来说，其实结合起来没什么问题，因为对于这种情况，祖先元素依然可以通过向上检索进行访问。但对于 full element transclusion 来说，就会遇到问题。在 element transclusion 指令所在的元素上，如果有其他指令含有控制器，如果我们尝试在 transclusion 内容中对这些控制器进行引入，就会发现根本没法做到：

_test/compile_spec.js_

```js
it('supports requiring controllers', function() {
  var MyController = function() {};
  var gotCtrl;
  var injector = makeInjectorWithDirectives({
    myCtrlDirective: function() {
      return {
        controller: MyController
      };
    },
    myTranscluder: function() {
      return {
        transclude: 'element',
        link: function(scope, el, attrs, ctrl, transclude) {
          el.after(transclude());
        }
      };
    },
    myOtherDirective: function() {
      return {
        require: '^myCtrlDirective',
        link: function(scope, el, attrs, ctrl, transclude) {
          gotCtrl = ctrl;
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div><div my-ctrl-directive my-transcluder><div my-other-directive></div></div>');

    $compile(el)($rootScope);

    expect(gotCtrl).toBeDefned();
    expect(gotCtrl instanceof MyController).toBe(true);
  });
});
```

这样做是因为含有控制器的元素会变成一个 HTML 注释节点，而注释节点不支持 jQuery data，并且它也不在经过 transclude 的内容的祖先中（相反，只是它的兄弟节点而已）。因此，当前的代码开发并不能解决这个问题。

我们要做就是显式地提供实际进行 transclude 的那个元素的控制器的引用。我们需要用 JavaScript 来干这个，因为 jQuery data 没办法用了。

首先，我们会在`applyDirectivesToNode`函数中声明一个新变量，这个变量用于标记当前节点是否带有 element transclusion 指令。

_src/compile.js_

```js
function applyDirectivesToNode(
  directives, compileNode, attrs, previousCompileContext) {
  // previousCompileContext = previousCompileContext || {};
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // var preLinkFns = previousCompileContext.preLinkFns || [];
  // var postLinkFns = previousCompileContext.postLinkFns || [];
  // var controllers = {};
  // var newScopeDirective;
  // var newIsolateScopeDirective = previousCompileContext.newIsolateScopeDirective;
  // var templateDirective = previousCompileContext.templateDirective;
  // var controllerDirectives = previousCompileContext.controllerDirectives;
  // var childTranscludeFn;
  // var hasTranscludeDirective = previousCompileContext.hasTranscludeDirective;
  var hasElementTranscludeDirective;
  
  // ...
}
```

当发现带有 element transclusion 指令时，我们就把标识赋值为`true`：

```js
if (directive.transclude === 'element') {
  hasElementTranscludeDirective = true;
  // var $originalCompileNode = $compileNode;
  // $compileNode = attrs.$$element = $(document.createComment(
  //   ' ' + directive.name + ': ' + attrs[directive.name] + ' '
  // ));
  // $originalCompileNode.replaceWith($compileNode);
  // terminalPriority = directive.priority;
  // childTranscludeFn = compile($originalCompileNode, terminalPriority);
} else {
  // ...
}
```

在绑定了作用域的 transclusion 函数中，我们会传递一个新参数给 inner bound transclusion function（但只有在进行 full element transclusion 才这样做）。这个参数保存的是包含当前元素上的所有控制器的对象：

```js
function scopeBoundTranscludeFn(transcludedScope, cloneAttachFn) {
  var transcludeControllers;
  // if (!transcludedScope || !transcludedScope.$watch ||
  //   !transcludedScope.$evalAsync) {
  //   cloneAttachFn = transcludedScope;
  //   transcludedScope = undefned;
  // }
  if (hasElementTranscludeDirective) {
    transcludeControllers = controllers;
  }
  return boundTranscludeFn(
    transcludedScope, cloneAttachFn, transcludeControllers, scope);
}
// scopeBoundTranscludeFn.$$boundTransclude = boundTranscludeFn;
```

在`boundTranscludeFn`中，我们现在需要接收这个参数，并把它传递给原始的 transclusion 函数（也就是 transclude 内容的公共链接函数），作为`options`对象的一部分：

```js
// var boundTranscludeFn;
// if (linkFn.nodeLinkFn.transcludeOnThisElement) {
  boundTranscludeFn = function(
    transcludedScope, cloneAttachFn, transcludeControllers, containingScope) {
    // if (!transcludedScope) {
    //   transcludedScope = scope.$new(false, containingScope);
    // }
    return linkFn.nodeLinkFn.transclude(transcludedScope, cloneAttachFn, {
      transcludeControllers: transcludeControllers
    });
  };
// } else if (parentBoundTranscludeFn) {
//   boundTranscludeFn = parentBoundTranscludeFn;
// }
```

最后，在公共链接函数中就可以从`options`对象中访问到任何一个 transclude 控制器，并把它们作为 jQuery data 附加到 transclude 节点上。这意味着，在这个含有 transclude 节点内部可以访问到所有相关控制器，通过`require`功能进行查找就可以了：

```js
return function publicLinkFn(scope, cloneAttachFn, options) {
  // options = options || {};
  // var parentBoundTranscludeFn = options.parentBoundTranscludeFn;
  // var transcludeControllers = options.transcludeControllers;
  // if (parentBoundTranscludeFn && parentBoundTranscludeFn.$$boundTransclude) {
  //   parentBoundTranscludeFn = parentBoundTranscludeFn.$$boundTransclude;
  // }
  // var $linkNodes;
  // if (cloneAttachFn) {
  //   $linkNodes = $compileNodes.clone();
  //   cloneAttachFn($linkNodes, scope);
  // } else {
  //   $linkNodes = $compileNodes;
  // }
  _.forEach(transcludeControllers, function(controller, name) {
    $linkNodes.data('$' + name + 'Controller', controller.instance);
  });
  // $linkNodes.data('$scope', scope);
  // compositeLinkFn(scope, $linkNodes, parentBoundTranscludeFn);
  // return $linkNodes;
};
```

回顾之前我们所学的，这个控制器对象包含的其实都是“非完全构造”的控制器函数，也就是在我们把实际的控制器对象附加到 DOM 之前，需要使用`instance`属性。