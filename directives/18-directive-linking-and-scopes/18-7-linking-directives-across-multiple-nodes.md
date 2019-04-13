### 对跨节点指令进行链接（Linking Directives Across Multiple Nodes）

上面章节中我们谈到了怎么对跨节点指令进行配置，并在 DOM 中通过`-start`和`-end`后缀进行应用。这种情况需要在链接时进行区分对待，因为我们希望的是这种指令可以接收了多个元素，但目前的代码实现只会接收到一个开始指令元素：

_test/compile_spec.js_

```js
it('invokes multi-element directive link functions with whole group', function() {
  var givenElements;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      multiElement: true,
      link: function(scope, element, attrs) {
        givenElements = element;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $(
      '<div my-directive-start></div>' +
      '<p></p>' +
      '<div my-directive-end></div>'
    );
    $compile(el)($rootScope);
    expect(givenElements.length).toBe(3);
  });
});
```

我们将要做的是对`applyDirectivesToNode`添加一些逻辑来处理跨元素的指令应用。但首先我们需要做一些重构，我们会引入一个帮助函数来收集节点的链接函数，这样对指令的`_.forEach`循环代码就会减少一点：

_src/compile.js_

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // var preLinkFns = [],
  //   postLinkFns = [];

  function addLinkFns(preLinkFn, postLinkFn) {
    if (preLinkFn) {
      preLinkFns.push(preLinkFn);
    }
    if (postLinkFn) {
      postLinkFns.push(postLinkFn);
    }
  }
  // _.forEach(directives, function(directive) {
  //   if (directive.$$start) {
  //     $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  //   }
  //   if (directive.priority < terminalPriority) {
  //     return false;
  //   }
  //   if (directive.compile) {
  //     var linkFn = directive.compile($compileNode, attrs);
  //     if (_.isFunction(linkFn)) {
        addLinkFns(null, linkFn);
      // } else if (linkFn) {
        addLinkFns(linkFn.pre, linkFn.post);
  //     }
  //   }
  //   if (directive.terminal) {
  //     terminal = true;
  //     terminalPriority = directive.priority;
  //   }
  // });
  // ...
}
```

这个新的`addLinkFns`函数将会包含一些处理跨节点的诀窍，但首先我们得让它能判别是否出现了跨节点的情况。之前我们已经针对跨节点的情况而对指令对象附加了`$$start`和`$$end`属性（这是在`addDirective`里按需附加的）。这两个属性保存了在 DOM 里面标志开始和结束的指令属性名。下面我们会将这两个属性传递到`addLinkFns`：

```js
_.forEach(directives, function(directive) {
  // if (directive.$$start) {
  //   $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  // }
  // if (directive.compile) {
  //   var linkFn = directive.compile($compileNode, attrs);
    var attrStart = directive.$$start;
    var attrEnd = directive.$$end;
    // if (_.isFunction(linkFn)) {
      addLinkFns(null, linkFn, attrStart, attrEnd);
    // } else if (linkFn) {
      addLinkFns(linkFn.pre, linkFn.post, attrStart, attrEnd);
  //   }
  // }
  // if (directive.terminal) {
  //   terminal = true;
  //   terminalPriority = directive.priority;
  // }
});
```

在`addLinkFns`函数中，我们会对传入的开始指令参数进行非空判断。如果不为空，我们就会把链接函数用一个特殊的帮助函数进行包裹，这个帮助函数就知道如何处理元素节点集的起始和结束：

```js
function addLinkFns(preLinkFn, postLinkFn, attrStart, attrEnd) {
  // if (preLinkFn) {
    if (attrStart) {
      preLinkFn = groupElementsLinkFnWrapper(preLinkFn, attrStart, attrEnd);
    }
  //   preLinkFns.push(preLinkFn);
  // }
  // if (postLinkFn) {
    if (attrStart) {
      postLinkFn = groupElementsLinkFnWrapper(postLinkFn, attrStart, attrEnd);
    }
  //   postLinkFns.push(postLinkFn);
  // }
}
```

这个`groupElementsLinkFnWrapper`函数会返回一个被“包裹”的链接函数，而这个包裹函数
实际上会做的就是用一个扫描后得到的元素节点集来代替原来的`element`。对于扫描元素节点集，我们可以利用之前实现的`groupScan`函数，`groupScan`函数之前也在编译阶段被用来做相同的事情：

_src/compile.js_

```js
function groupScan(node, startAttr, endAttr) {
  // ..
}

function groupElementsLinkFnWrapper(linkFn, attrStart, attrEnd) {
  return function(scope, element, attrs) {
    var group = groupScan(element[0], attrStart, attrEnd);
    return linkFn(scope, group, attrs);
  };
}
```

所以，这个包裹函数其实是公共链接函数和指令链接函数之间的一个额外步骤，只要传递了开始指令元素、开始指令名称和结束指令名称，它就懂得处理元素集。