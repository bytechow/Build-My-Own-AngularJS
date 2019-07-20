### 保存独立作用域指令（Preserving The Isolate Scope Directive）

还有一个被我们“遗忘”的情况，就是在异步加载模板时，同一元素上还存在一个独立作用域指令的情况。如果存在的话，链接会失败：

_test/compile_spec.js_

```js
it('retains isolate scope directives from earlier', function() {
  var linkSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: {
          val: '=myDirective'
        },
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
    var el = $('<div my-directive="42" my-other-directive></div>');
    
    var linkFunction = $compile(el);
    $rootScope.$apply();
    
    linkFunction($rootScope);
    
    requests[0].respond(200, {}, '<div></div>');

    expect(linkSpy).toHaveBeenCalled();
    expect(linkSpy.calls.frst().args[0]).toBeDefned();
    expect(linkSpy.calls.frst().args[0]).not.toBe($rootScope);
    expect(linkSpy.calls.frst().args[0].val).toBe(42);
  });
});
```

这也是我们需要保存在`previousCompileContext`里面的东西：

_src/compile.js_

```js
nodeLinkFn = compileTemplateUrl(
  // _.drop(directives, i),
  // $compileNode,
  // attrs, 
  {
    // templateDirective: templateDirective,
    newIsolateScopeDirective: newIsolateScopeDirective,
    // preLinkFns: preLinkFns,
    // postLinkFns: postLinkFns
  }
);
```

相应地，我们需要在二次调用`applyDirectivesToNode`时把这个值翻出来：

```js
function applyDirectivesToNode(
  directives, compileNode, attrs, previousCompileContext) {
  previousCompileContext = previousCompileContext || {};
  var $compileNode = $(compileNode);
  var terminalPriority = -Number.MAX_VALUE;
  var terminal = false;
  var preLinkFns = previousCompileContext.preLinkFns || [];
  var postLinkFns = previousCompileContext.postLinkFns || [];
  var controllers = {};
  var newScopeDirective;
  var newIsolateScopeDirective = previousCompileContext.newIsolateScopeDirective;
  var templateDirective = previousCompileContext.templateDirective;
  var controllerDirectives;
```

这里还有一个没解决的问题，会出现在独立作用域和模板 URL 出现在同一个指令上时：

_test/compile_spec.js_

```js
it('supports isolate scope directives with templateUrls', function() {
  var linkSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: {
          val: '=myDirective'
        },
        link: linkSpy,
        templateUrl: '/my_other_directive.html'
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive="42"></div>');

    var linkFunction = $compile(el)($rootScope);
    $rootScope.$apply();
    
    requests[0].respond(200, {}, '<div></div>');
    
    expect(linkSpy).toHaveBeenCalled();
    expect(linkSpy.calls.frst().args[0]).not.toBe($rootScope);
    expect(linkSpy.calls.frst().args[0].val).toBe(42);
  });
});
```

链接函数现在并没有被调用！这是因为在异步模板加载的过程中发生了一个异常（不幸的是，我们没法看到这个异常）

发生错误的根本原因是我们是在一个错误的指令上设置我们的独立作用域绑定。在我们开始异步加载之前，是在异步的指令上初始化绑定，其实我们只应该在派生的同步指令进行初始化的（？）。

这其实也很容易解决的，我们对是新作用域或者独立作用域，但带有模板 URL 的情况进行拦截。我们会在指令加载过来的时候再对这种指令进行处理：

_src/compile.js_

```js
if (directive.scope && !directive.templateUrl) {
  if (_.isObject(directive.scope)) {
    if (newIsolateScopeDirective || newScopeDirective) {
      throw 'Multiple directives asking for new/inherited scope';
    }
    newIsolateScopeDirective = directive;
  } else {
    if (newIsolateScopeDirective) {
      throw 'Multiple directives asking for new/inherited scope';
    }
    newScopeDirective = newScopeDirective || directive;
  }
}
``` 

这时另一个问题就浮出水面了，是在我们检查独立作用域指令模板里面的指令链接时出现的。本章前面我们介绍了它们应该使用独立作用域进行链接，因为独立作用域指令“拥有”这些指令。但如果是在异步加载模板的情况下，事情就发生改变了。

```js
it('links children of isolate scope directives with templateUrls', function() {
  var linkSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: {
          val: '=myDirective'
        },
        templateUrl: '/my_other_directive.html'
      };
    },
    myChildDirective: function() {
      return {
        link: linkSpy
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive="42"></div>');
    var linkFunction = $compile(el)($rootScope);
    $rootScope.$apply();
    requests[0].respond(200, {}, '<div my-child-directive></div>');
    expect(linkSpy).toHaveBeenCalled();
    expect(linkSpy.calls.frst().args[0]).not.toBe($rootScope);
    expect(linkSpy.calls.frst().args[0].val).toBe(42);
  });
});
```

子指令实际上是链接到根作用域，而不是独立作用域。

对于这个问题，我们能在节点链接函数中找到“始作俑者”，在节点链接函数中我们会决定用哪个作用域对节点的子元素进行链接。我们会检查独立作用域指令是不是带有`template`属性，只有在这种情况下，我们才会使用独立作用域进行链接。派生的同步指令并没有模板，也没有模板 URL。但它同样需要用独立作用域来链接子节点！

要修复这个问题，我们会利用一个事实，就是我们会对派生的同步指令的`templateUrl`显式赋值为`null`。在节点链接函数中，我们会检查看独立作用域指令是否有`template`属性，或者看它的`templateUrl`属性是否被赋值为`null`。这样我们就满足了需要链接到独立作用域的两种情况了。

_src/compile.js_

```js
if (childLinkFn) {
  // var scopeToChild = scope;
  if (newIsolateScopeDirective &&
    (newIsolateScopeDirective.template ||
      newIsolateScopeDirective.templateUrl === null)) {
    // scopeToChild = isolateScope;
  }
  // childLinkFn(scopeToChild, linkNode.childNodes);
}
```