### 带模板 URL 的 transclusion（Transclusion with Tempalte URLs）

在上一章，我们花费了很大的力气建立一套可以暂停和重启的机制来实现异步模板的加载。那这个机制能兼容有 transclusion 的情况吗？

目前的答案会是不能。当使用了`templateUrls`是，节点链接函数目前接收到的绑定 transclusion 函数并不被这套机制里的_延迟的节点链接函数_所支持。但是否使用 transclusion 不应该影响使用`templateUrls`。

让我们先考虑一下使用了`templateUrl`但在链接开始之前仍然接收到模板的情况。让我们在`compile_spec.js`的“templateUrl”部分加入下面这个测试用例：

_test/compile_spec.js_

```js
describe('with transclusion', function() {

  it('works when template arrives frst', function() {
    var injector = makeInjectorWithDirectives({
      myTranscluder: function() {
        return {
          transclude: true,
          templateUrl: 'my_template.html',
          link: function(scope, element, attrs, ctrl, transclude) {
            element.find('[in-template]').append(transclude());
          }
        };
      }
    });
    injector.invoke(function($compile, $rootScope) {
      var el = $('<div my-transcluder><div in-transclude></div></div>');

      var linkFunction = $compile(el);
      $rootScope.$apply();
      requests[0].respond(200, {}, '<div in-template></div>'); // respond frst
      linkFunction($rootScope); // then link
      
      expect(el.find('> [in-template] > [in-transclude]').length).toBe(1);
    });
  });
  
});
```

正如所料，这个单元测试失败了。我们需要做的是，当传递给让_延迟的节点链接函数_一个绑定的 transclusion 函数时，它可以接收并处理。如果模板已经获取到了，而`linkQueue`不再存在，我们可以把它（绑定的 transclusion 函数）传递给普通的节点链接函数：

_src/compile.js_

```js
return function delayedNodeLinkFn(
  _ignoreChildLinkFn, scope, linkNode, boundTranscludeFn) {
  if (linkQueue) {
    linkQueue.push({
      scope: scope,
      linkNode: linkNode
    });
  } else {
    afterTemplateNodeLinkFn(
      afterTemplateChildLinkFn, scope, linkNode, boundTranscludeFn);
  }
};
```

但单元测试还是没能通过。发生了什么？

问题在于这个指令加上了`transclude`选项，一旦我们对模板里的内容进行插入，并在`compileTemplateUrl`里面调用`applyDirectivesToNode`了，所有来自模板的节点都会被清除掉。我们不希望在接收到模板后再做这些事情，毕竟我们已经在上一次调用`applyDirectivesToNode`时设置了 transclusion。所以我们会在出来的同步指令中把`transclude`标识设置为`null`，这样的话 transclusion 的相关逻辑就不会被触发两次了：

```js
function compileTemplateUrl(
  directives, $compileNode, attrs, previousCompileContext) {
  var origAsyncDirective = directives.shift();
  var derivedSyncDirective = _.extend(
    {},
    origAsyncDirective, {
      templateUrl: null,
      transclude: null
    }
  );
  // ...
}
```

对于需要请求模板 URL的情况，我们还没解决的另一个问题是，在模板从服务器中成功传送过来之前，我们会在什么时候调用公共链接函数。在这种情况下，我们依然没能正确处理 transclusion：

_test/compile_spec.js_

```js
it('works when template arrives after', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true,
        templateUrl: 'my_template.html',
        link: function(scope, element, attrs, ctrl, transclude) {
          element.find('[in-template]').append(transclude());
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div in-transclude></div></div>');

    var linkFunction = $compile(el);
    $rootScope.$apply();
    linkFunction($rootScope); // link first
    requests[0].respond(200, {}, '<div in-template></div>'); // then respond
    
    expect(el.find('> [in-template] > [in-transclude]').length).toBe(1);
  });
});
```

这种情况需要用到用于存储未执行链接的链接任务队列中处理这种情况。我们需要把`boundTranscludeFn`传递到链接队列元素中，这样我们才不会丢失它：

_src/compile.js_

```js
return function delayedNodeLinkFn(
  _ignoreChildLinkFn, scope, linkNode, boundTranscludeFn) {
  if (linkQueue) {
    linkQueue.push({
      // scope: scope,
      // linkNode: linkNode,
      boundTranscludeFn: boundTranscludeFn
    });
  } else {
    // afterTemplateNodeLinkFn(
    //   afterTemplateChildLinkFn, scope, linkNode, boundTranscludeFn);
  }
};
```

一旦接收到模板，我们就可以拿到之前保存的`boundTranscludeFn`，并把它传递给节点链接函数：

```js
$http.get(templateUrl).success(function(template) {
  // directives.unshift(derivedSyncDirective);
  // $compileNode.html(template);
  // afterTemplateNodeLinkFn = applyDirectivesToNode(
  //   directives, $compileNode, attrs, previousCompileContext);
  // afterTemplateChildLinkFn = compileNodes($compileNode[0].childNodes);
  _.forEach(linkQueue, function(linkCall) {
    afterTemplateNodeLinkFn(
      // afterTemplateChildLinkFn,
      // linkCall.scope,
      // linkCall.linkNode,
      linkCall.boundTranscludeFn
    );
  });
  // linkQueue = null;
});
```

关于异步模板与 transclusion，还有最后一个问题需要解决，对在同一个元素上的两个指令都使用了 transclusion 进行控制。我们已经对普通的同步指令进行过这个检查，但还没有兼容有`templateUrl`的情况。这种情况下，第二个是用了 transclusion 指令就不应该被编译了：

_test/compile_spec.js_

```js
it('is only allowed once', function() {
  var otherCompileSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        priority: 1,
        transclude: true,
        templateUrl: 'my_template.html'
      };
    },
    mySecondTranscluder: function() {
      return {
        priority: 0,
        transclude: true,
        compile: otherCompileSpy
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder my-second-transcluder></div>');

    $compile(el);
    $rootScope.$apply();
    requests[0].respond(200, {}, '<div in-template></div>');
    
    expect(otherCompileSpy).not.toHaveBeenCalled();
  });
})
```

我们需要在构建`previousCompileContextObject`时把`hasTranscludeDirective`跟踪变量也传递过去：

_src/compile.js_

```js
nodeLinkFn = compileTemplateUrl(
  _.drop(directives, i),
  $compileNode,
  attrs, {
    // templateDirective: templateDirective,
    // newIsolateScopeDirective: newIsolateScopeDirective,
    // controllerDirectives: controllerDirectives,
    hasTranscludeDirective: hasTranscludeDirective,
    // preLinkFns: preLinkFns,
    // postLinkFns: postLinkFns
  }
);
```

然后我们只需要在`applyDirectivesToNode`把它拿出来就可以了：

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
  var hasTranscludeDirective = previousCompileContext.hasTranscludeDirective;
  
  // ...
}
```