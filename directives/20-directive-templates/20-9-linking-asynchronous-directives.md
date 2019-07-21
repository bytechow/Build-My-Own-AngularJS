### 链接异步指令（Linking Asynchronous Directives）

现在我们可以在异步获取模板后全面重启编译过程了，但我们现在还是缺失对已经异步编译好的指令进行链接的功能。现在我们的做法是简单地丢弃它们的链接函数，这是因为当`applyDirectivesToNode`二次调用时，它返回的节点链接函数会被丢弃。这意味着这些指令将永远不会被调用。

但事情不应该这样处理。当公共的链接函数被调用时，已经异步编译好的指令应该能够像其他指令一样进行链接：

_test/compile_spec.js_

```js
it('links the directive when public link function is invoked', function() {
  var linkSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html',
        link: linkSpy
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');

    var linkFunction = $compile(el);
    $rootScope.$apply();
    
    requests[0].respond(200, {}, '<div></div>');
    
    linkFunction($rootScope);
    expect(linkSpy).toHaveBeenCalled();
    expect(linkSpy.calls.frst().args[0]).toBe($rootScope);
    expect(linkSpy.calls.frst().args[1][0]).toBe(el[0]);
    expect(linkSpy.calls.frst().args[2].myDirective).toBeDefned();
  });
});
```

模板里面的子元素也是一样的。我们希望它们也会被链接，但现在它们还是没有被链接。这是因为我们在`compileTemplateUrl`里调用`compileNodes()`的返回也被丢弃了。

_test/compile_spec.js_

```js
it('links child elements when public link function is invoked', function() {
  var linkSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html'
      };
    },
    myOtherDirective: function() {
      return {
        link: linkSpy
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');

    var linkFunction = $compile(el);
    $rootScope.$apply();
    
    requests[0].respond(200, {}, '<div my-other-directive></div>');
    
    linkFunction($rootScope);
    expect(linkSpy).toHaveBeenCalled();
    expect(linkSpy.calls.frst().args[0]).toBe($rootScope);
    expect(linkSpy.calls.frst().args[1][0]).toBe(el[0].frstChild);
    expect(linkSpy.calls.frst().args[2].myOtherDirective).toBeDefned();
  });
});
```

我们启用异步链接的过程分为三步，下面我们就逐个进行分析。

`applyDirectivesToNode`约定会返回节点的链接函数。节点链接函数是在`applyDirectivesToNode`中通过语句`function
nodeLinkFn(...)`进行定义的。

当节点上有一个指令需要加载异步模板时，我们就不应该再返回一个普通的节点链接函数了，而是一个特殊的“延迟的节点链接函数”。我们会设置成用`compileTemplateUrl`返回“延迟的节点链接函数”。假如真如我们所说的，我们就可以捕捉返回值，并重写局部变量`nodeLinkFn`。最后，“延迟的节点链接函数”会成为`applyDirectivesToNode`的返回值：

_src/compile.js_

```js
if (directive.templateUrl) {
  // if (templateDirective) {
  //   throw 'Multiple directives asking for template';
  // }
  // templateDirective = directive;
  nodeLinkFn = compileTemplateUrl(
    _.drop(directives, i),
    $compileNode,
    attrs, {
      templateDirective: templateDirective
    }
  );
  // return false;
```

在`compileTemplateUrl`里面，我们现在就需要引入这个延迟的节点连接函数。这个函数将负责处理这个节点的所有链接过程，由于取代了原先的节点链接函数，这个函数担起这个责任。这样处理的最后结果是，当调用了这个节点的节点链接函数，实际上就是调用了`delayedNodeLinkFn`：

```js
function compileTemplateUrl(
  directives, $compileNode, attrs, previousCompileContext) {
  // var origAsyncDirective = directives.shift();
  // var derivedSyncDirective = _.extend({},
  //   origAsyncDirective, {
  //     templateUrl: null
  //   }
  // );
  // var templateUrl = _.isFunction(origAsyncDirective.templateUrl) ?
  //   origAsyncDirective.templateUrl($compileNode, attrs) :
  //   origAsyncDirective.templateUrl;
  // $compileNode.empty();
  // $http.get(templateUrl).success(function(template) {
  //   directives.unshift(derivedSyncDirective);
  //   $compileNode.html(template);
  //   applyDirectivesToNode(
  //     directives, $compileNode, attrs, previousCompileContext);
  //   compileNodes($compileNode[0].childNodes);
  // });
  return function delayedNodeLinkFn() {
    
  };
}
```

那我们应该在延迟的节点链接函数里面做什么呢？首先我们必须要做的一件事，就是把我们异步编译过的所有指令进行链接，包括当前节点以及它的子节点的。我们是通过之前抛出的、分别调用`applyDirectivesToNode`和`compileNodes`获取的返回值，来获取链接函数的。我们把它们称为`afterTemplateNodeLinkFn`和`afterTemplateChildLinkFn`，因为它们都是在加载了模板后再进行链接的链接函数：

```js
function compileTemplateUrl(
  directives, $compileNode, attrs, previousCompileContext) {
  // var origAsyncDirective = directives.shift();
  // var derivedSyncDirective = _.extend({},
  //   origAsyncDirective, {
  //     templateUrl: null
  //   }
  // );
  // var templateUrl = _.isFunction(origAsyncDirective.templateUrl) ?
  //   origAsyncDirective.templateUrl($compileNode, attrs) :
  //   origAsyncDirective.templateUrl;
  var afterTemplateNodeLinkFn, afterTemplateChildLinkFn;
  // $compileNode.empty();
  // $http.get(templateUrl).success(function(template) {
  //   directives.unshift(derivedSyncDirective);
  //   $compileNode.html(template);
    afterTemplateNodeLinkFn = applyDirectivesToNode(
      directives, $compileNode, attrs, previousCompileContext);
    afterTemplateChildLinkFn = compileNodes($compileNode[0].childNodes);
  // });
  // return function delayedNodeLinkFn() {
  // 
  // };
}
```