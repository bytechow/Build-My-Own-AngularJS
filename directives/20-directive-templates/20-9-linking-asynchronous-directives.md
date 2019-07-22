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

我们需要从延迟的节点链接函数调用这两个函数。但先让我们来考虑一个问题，就是延迟的节点链接函数会接收的参数。它会被作为一个普通的节点链接函数进行调用，也就是说它会接收到三个参数：

1. 子节点的链接函数
2. 需要进行链接的作用域
3. 需要进行链接的节点

后面两个参数是不需要再加以说明的，但第一个参数，也就是子节点的链接函数就比较有趣了。它将是模板加载完成之前的子节点链接函数。由于我们会在加载模板之前清空节点原来的子节点，所以实际上这个函数并没有做什么事情。我们可以在编译时安全地忽略它，转而使用`afterTemplateChlidLinkFn`：

_src/compile.js_

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
  // var afterTemplateNodeLinkFn, afterTemplateChildLinkFn;
  // $compileNode.empty();
  // $http.get(templateUrl).success(function(template) {
  //   directives.unshift(derivedSyncDirective);
  //   $compileNode.html(template);
  //   afterTemplateNodeLinkFn = applyDirectivesToNode(
  //     directives, $compileNode, attrs, previousCompileContext);
  //   afterTemplateChildLinkFn = compileNodes($compileNode[0].childNodes);
  // });
  
  return function delayedNodeLinkFn(_ignoreChildLinkFn, scope, linkNode) {
    afterTemplateNodeLinkFn(afterTemplateChildLinkFn, scope, linkNode);
  };
}
```

这样确实满足了单元测试的要求，但如果你仔细留意一下测试用例中代码执行的完成顺序，你可能会发现有一点奇怪：测试中是等接收到模板后再调用公共链接函数的。我们的代码实现也应该要满足这个顺序。

这并不是一个合理的需求。如果保持目前的处理，调用公共的链接函数的人，需要在调用之前确保所有的异步模板获取过程是否都结束了。实际上，作为一个 Angular 使用者是不需要考虑这个问题的。所以实际上在编译结束后立即调用链接函数是更普遍的做法。

所以我们需要处理在接收到异步模板之前公共链接函数就被链接的情况，也就是在创建`afterTemplateNodeLinkFn`和`afterTemplateChildLinkFn`之前。我们要保证链接会发生在最终接收到模板之后：

_test/compile_spec.js_

```js
it('links when template arrives if node link fn was called', function() {
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

    var linkFunction = $compile(el)($rootScope); // link frst
    
    $rootScope.$apply();
    requests[0].respond(200, {}, '<div></div>'); // then receive template
    
    expect(linkSpy).toHaveBeenCalled();
    expect(linkSpy.calls.argsFor(0)[0]).toBe($rootScope);
    expect(linkSpy.calls.argsFor(0)[1][0]).toBe(el[0]);
    expect(linkSpy.calls.argsFor(0)[2].myDirective).toBeDefned();
  });
});
```

这就意味着在我们开始链接之前，`delayedNodeLinkFn`可能已经被调用了。如果是这样的话，我们需要对调用时的参数进行保存，以便到准备就绪时再调用。我们会把这些参数保存到`compileTemplateUrl`里面的“链接队列”里。我们会把它初始化为数组，当我们接收到这个模板时再把它设置为`null`：

_src/compile.js_

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
  // var afterTemplateNodeLinkFn, afterTemplateChildLinkFn;
  var linkQueue = [];
  // $compileNode.empty();
  // $http.get(templateUrl).success(function(template) {
  //   directives.unshift(derivedSyncDirective);
  //   $compileNode.html(template);
  //   afterTemplateNodeLinkFn = applyDirectivesToNode(
  //     directives, $compileNode, attrs, previousCompileContext);
  //   afterTemplateChildLinkFn = compileNodes($compileNode[0].childNodes);
    linkQueue = null;
  // });
  
  // return function delayedNodeLinkFn(_ignoreChildLinkFn, scope, linkNode) {
  //   afterTemplateNodeLinkFn(afterTemplateChildLinkFn, scope, linkNode);
  // };
}
```

现在在`delayedNodeLinkFn`我们有两种选择：如果是存在一个链接队列的话，就把参数放在那里就好，因为它们都还没就绪。而如果没有链接队列（因为这时它已经被设置为`null`了），就直接像之前那样调用节点链接函数：

_src/compile.js_

```js
return function delayedNodeLinkFn(_ignoreChildLinkFn, scope, linkNode) {
  if (linkQueue) {
    linkQueue.push({ scope: scope, linkNode: linkNode});
  } else {
    // afterTemplateNodeLinkFn(afterTemplateChildLinkFn, scope, linkNode);
  }
};
```

现在，当我们接收到模板时，如果链接函数已经准备好被调用的话，那链接队列中就会有一个或更多元素。我们将会运用那些调用对象：

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
  // var afterTemplateNodeLinkFn, afterTemplateChildLinkFn;
  // var linkQueue = [];
  // $compileNode.empty();
  // $http.get(templateUrl).success(function(template) {
  //   directives.unshift(derivedSyncDirective);
  //   $compileNode.html(template);
  //   afterTemplateNodeLinkFn = applyDirectivesToNode(
  //     directives, $compileNode, attrs, previousCompileContext);
  //   afterTemplateChildLinkFn = compileNodes($compileNode[0].childNodes);
    _.forEach(linkQueue, function(linkCall) {
      afterTemplateNodeLinkFn(
        afterTemplateChildLinkFn, linkCall.scope, linkCall.linkNode);
    });
    // linkQueue = null;
  // });
  // return function delayedNodeLinkFn(_ignoreChildLinkFn, scope, linkNode) {
  //   if (linkQueue) {
  //     linkQueue.push({
  //       scope: scope,
  //       linkNode: linkNode
  //     });
  //   } else {
  //     afterTemplateNodeLinkFn(afterTemplateChildLinkFn, scope, linkNode);
  //   }
  // };
}
```

> 通常链接函数只会被调用一次，所以确实没必要为多个链接函数的调用建立队列。但因为技术上允许多次调用同一个链接函数，所以这个功能也因异步模板而得以保留。

因为链接队列实际上会随着 DOM 子树的链接进程而发生改变，所以它只会在异步模板加载完成时进行初始化。

注意着也意味着当你在 Angular 中调用一个公共链接函数，当这个函数被返回时，你并不能保证所有的链接过程都完成了。如果 DOM 中有带异步模板的指令，链接过程只可能在模板加载完成后完成。

