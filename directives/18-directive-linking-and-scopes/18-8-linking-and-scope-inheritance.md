### 链接过程与作用域继承（Linking And Scope Inheritance）

了解了链接进行的基本步骤后，我们继续介绍这个章节的另一个重要主题：在指令链接时新的作用域如果构建出来。

目前我们的代码中会对公共链接函数传递作用域作为参数，并把这个作用域也作为所有指令链接函数的作用域。也就是说在 DOM 树上所有指令都共享一个作用域。虽然这种情况在 Angular 应用中确实存在，但更常见的情况时各个指令都需要有自己的一个作用域，这就要用到我们在本书第二章谈到的继承机制。让我们来看看这是怎么实现的。

一个指令可能会要求有自己的一个继承作用域，我们希望可以通过在指令定义对象上声明`scope`属性为`true`来实现：

_test/compile_spec.js_

```js
it('makes new scope for element when directive asks for it', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: true,
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(givenScope.$parent).toBe($rootScope);
  });
});
```

> 如果我们传入的`scope`属性为`false`，那 Angular 会把这个值等同于`undefined`。也就是说，这时指令应该从它的上下文环境中接收作用域，这就是我们目前已经实现的效果了。

只要元素上有一个指令需要继承作用域，这个元素上的其他指令接收的也会是这个继承作用域：

```js
it('gives inherited scope to all directives on element', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: true
      };
    },
    myOtherDirective: function() {
      return {
        link: function(scope) {
          givenScope = scope;
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    $compile(el)($rootScope);
    expect(givenScope.$parent).toBe($rootScope);
  });
});
```

这里我们对一个元素应用两个指令，其中一个指令请求了一个继承作用域。另一个指令不会主动请求继承作用域，我们会检查该指令是否也是会接收一个继承作用域。

当一个元素中需要继承作用域，这个元素就会附加上以下两样东西：

- 一个`ng-scope`的 CSS 样式类
- 一个新的作用域对象，会作为 jQuery/jqLite 数据属性

```js
it('adds scope class and data for element with new scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: true,
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(el.hasClass('ng-scope')).toBe(true);
    expect(el.data('$scope')).toBe(givenScope);
  });
});
```

下面我们就要让以上单元测试能够通过。首先要做的是对指令进行检测，看它是否需要一个新的作用域。在`applyDirectivesToNode`函数中，如果我们发现有一个指令的`scope`属性值为`true`，就对节点链接函数也设置一个相同的`scope`属性：

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // var preLinkFns = [],
  //   postLinkFns = [];
  var newScopeDirective;

  // function addLinkFns(preLinkFn, postLinkFn, attrStart, attrEnd) {
  //   // ...
  // }
  // _.forEach(directives, function(directive) {
  //   if (directive.$$start) {
  //     $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  //   }
  //   if (directive.priority < terminalPriority) {
  //     return false;
  //   }
    if (directive.scope) {
      newScopeDirective = newScopeDirective || directive;
    }
  //   if (directive.compile) {
  //     var linkFn = directive.compile($compileNode, attrs);
  //     var attrStart = directive.$$start;
  //     var attrEnd = directive.$$end;
  //     if (_.isFunction(linkFn)) {
  //       addLinkFns(null, linkFn, attrStart, attrEnd);
  //     } else if (linkFn) {
  //       addLinkFns(linkFn.pre, linkFn.post, attrStart, attrEnd);
  //     }
  //   }
  //   if (directive.terminal) {
  //     terminal = true;
  //     terminalPriority = directive.priority;
  //   }
  // });

  // function nodeLinkFn(childLinkFn, scope, linkNode) {
  //   // ...
  // }
  // nodeLinkFn.terminal = terminal;
  nodeLinkFn.scope = newScopeDirective && newScopeDirective.scope;
  
  // return nodeLinkFn;
}
```

在复合链接函数中，如果节点链接函数标识为需要一个新的作用域，我们就为它创建一个新的作用域，也就是说，这个节点至少含有一个指令是需要继承作用域的：

```js
_.forEach(linkFns, function(linkFn) {
  // if (linkFn.nodeLinkFn) {
    if (linkFn.nodeLinkFn.scope) {
      scope = scope.$new();
    }
  //   linkFn.nodeLinkFn(
  //     linkFn.childLinkFn,
  //     scope,
  //     stableNodeList[linkFn.idx]
  //   );
  // } else {
  //   linkFn.childLinkFn(
  //     scope,
  //     stableNodeList[linkFn.idx].childNodes
  //   );
  // }
});
```

我们还需要对需要继承作用域的节点元素附加样式类和数据属性。实际上，CSS 样式类并不是在链接阶段加上的，而是在编译阶段就已经加上了的：

```js
_.forEach($compileNodes, function(node, i) {
  // var attrs = new Attributes($(node));
  // var directives = collectDirectives(node, attrs);
  // var nodeLinkFn;
  // if (directives.length) {
  //   nodeLinkFn = applyDirectivesToNode(directives, node, attrs);
  // }
  // var childLinkFn;
  // if ((!nodeLinkFn || !nodeLinkFn.terminal) &&
  //   node.childNodes && node.childNodes.length) {
  //   childLinkFn = compileNodes(node.childNodes);
  // }
  if (nodeLinkFn && nodeLinkFn.scope) {
    attrs.$$element.addClass('ng-scope');
  }
  // if (nodeLinkFn || childLinkFn) {
  //   linkFns.push({
  //     nodeLinkFn: nodeLinkFn,
  //     childLinkFn: childLinkFn,
  //     idx: i
  //   });
  // }
});
```

而以`$scope`命名的 jQuery 数据属性则是在链接阶段添加的，因为我们要到链接时才创建好新的作用域对象：

```js
_.forEach(linkFns, function(linkFn) {
  var node = stableNodeList[linkFn.idx];
  // if (linkFn.nodeLinkFn) {
  //   if (linkFn.nodeLinkFn.scope) {
  //     scope = scope.$new();
      $(node).data('$scope', scope);
    // }
    // linkFn.nodeLinkFn(
    //   linkFn.childLinkFn,
    //   scope,
      node
  //   );
  // } else {
  //   linkFn.childLinkFn(
  //     scope,
      node.childNodes
  //   );
  // }
});
```

就这样，我们就能通过单元测试了！

在本书的第一部分中，我们讨论过作用域的继承层次一般与 DOM 树的层次结构相匹配。现在，我们终于可以看到这是如何实现的。指令可以请求创建一个新的作用域，不论是应用该指令的元素节点还是它的子节点，都能获取一个继承作用域。