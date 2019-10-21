### 指令链接函数（Directive Link Functions）

如果链接函数做的仅仅是把`$scope`绑定到元素上，事情将不会那么地有趣。但显然它要做的东西更多。链接函数的主要任务还是在指令和 DOM 之间建立实际的链接。这也是为什么它称为_指令链接函数_的原因。

每个指令都可以拥有自己的链接函数。如果你曾经编写过自定义的指令，你会知道有链接函数的存在，而且也知道链接函数会被经常使用。

指令链接函数和编译函数有两个重要的区别：

1. 调用的时间点不同。指令编译函数会在编译时调用，而链接函数会在链接时被调用。区别主要与一些指令在这两个步骤会做的事情有关。比如，对于像`ngRepeat`这类会改变 DOM 的指令，你的指令会被编译一次，但会对每一个遍历的元素分别启动一次的链接。

2. 我们之前已经看到了，编译函数能够访问到 DOM 元素和属性对象。不过链接函数不止能访问到这些变量，还能访问到作用域对象。这里常常也是附加上应用数据和功能的地方。

定义链接函数有几种方式。我们直接先介绍最低层次的一种：当指令有`compile`函数时，它期望这个函数返回一个链接函数。我们来创建一个这样的的单元测试，并检查这个链接函数会接收到哪些参数：

_test/compile_spec.js_

```js
it('calls directive link function with scope', function() {
  var givenScope, givenElement, givenAttrs;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function() {
        return function link(scope, element, attrs) {
          givenScope = scope;
          givenElement = element;
          givenAttrs = attrs;
        };
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(givenScope).toBe($rootScope);
    expect(givenElement[0]).toBe(el[0]);
    expect(givenAttrs).toBeDefned();
    expect(givenAttrs.myDirective).toBeDefned();
  });
});
```

这个链接函数会接收三个参数：

1. 作用域对象，需要跟调用时传入的一致。
2. 一个元素，需要跟调用 compile 函数时传入的一致。
3. 属于要编译的元素的参数对象。

指令 API 的复杂性常被人诟病，但这些指责也通常有其道理。但指令 API 也有一种对称的美：就像公共的编译函数的返回值时一个公共的链接函数，一个指令的编译函数返回值同样是它的链接函数。这个模式在所有的编译和链接过程中被重复着。

为了通过测试，我们会先将公共的链接函数和指令的链接函数进行连接。在这两个过程之间，我们还会进行几个中间步骤。

公共的`compile`函数会调用`compileNodes`函数，它会编译一个节点集。这是第一个中间步骤：`compileNodes`函数应该给我们返回另一个链接函数。我们会把这个函数称为_复合链接函数_，因为它各个单独节点的链接函数的集合。这个复合链接函数会在公共的链接函数中被调用：

_src/compile.js_

```js
function compile($compileNodes) {
  var compositeLinkFn = compileNodes($compileNodes);

  return function publicLinkFn(scope) {
    $compileNodes.data('$scope', scope);
    compositeLinkFn(scope, $compileNodes);
  };
}
```

复合链接函数会接收两个参数：要链接的作用域，还有要链接的 DOM 元素。后者就是我们要编译的元素节点，但也并不一直这样，下面将会看到。

所以，在`compileNodes`我们会引入复合链接函数，并进行返回：

_src/compile.js_

```js
function compileNodes($compileNodes) {
  // _.forEach($compileNodes, function(node) {
  //   var attrs = new Attributes($(node));
  //   var directives = collectDirectives(node, attrs);
  //   var terminal = applyDirectivesToNode(directives, node, attrs);
  //   if (!terminal && node.childNodes && node.childNodes.length) {
  //     compileNodes(node.childNodes);
  //   }
  // });

  function compositeLinkFn(scope, linkNodes) {

  }

  return compositeLinkFn;
}
```

复合链接函数的工作是把对所有节点分别进行链接。对于每个节点来说，还有需要另一个层级的链接函数：每一个节点会拥有它们自己的一个节点链接函数，这个函数是由`applyDirectivesToNode`返回的.

注意，这意味着`applyDirectivesToNode`不能只返回`terminal`变量。我们会把`terminal`变量变成节点链接函数的一个属性：

```js
function compileNodes($compileNodes) {
  // _.forEach($compileNodes, function(node) {
  //   var attrs = new Attributes($(node));
  //   var directives = collectDirectives(node, attrs);
    var nodeLinkFn;
    if (directives.length) {
      nodeLinkFn = applyDirectivesToNode(directives, node, attrs);
    }
    if ((!nodeLinkFn || !nodeLinkFn.terminal) &&
  //     node.childNodes && node.childNodes.length) {
  //     compileNodes(node.childNodes);
  //   }
  // });

  // function compositeLinkFn(scope, linkNodes) {

  // }
  
  // return compositeLinkFn;
}
```

我们在各个节点编译时会用一个数组将它们的链接函数保存起来，同时也会保存当前该节点在节点集的索引：

```js
function compileNodes($compileNodes) {
  var linkFns = [];
  _.forEach($compileNodes, function(node, i) {
    // var attrs = new Attributes($(node));
    // var directives = collectDirectives(node, attrs);
    // var nodeLinkFn;
    // if (directives.length) {
    //   nodeLinkFn = applyDirectivesToNode(directives, node, attrs);
    // }
    // if ((!nodeLinkFn || !nodeLinkFn.terminal) &&
    //   node.childNodes && node.childNodes.length) {
    //   compileNodes(node.childNodes);
    // }
    if (nodeLinkFn) {
      linkFns.push({
        nodeLinkFn: nodeLinkFn,
        idx: i
      });
    }
  // });

  // function compositeLinkFn(scope, linkNodes) {

  // }
  
  // return compositeLinkFn;
}
```

这样在遍历节点之后，我们就可以获得一个对象数组，这个数组包含各个节点的链接函数和各自的索引。当然，我们只会对有指令的元素进行收集。

在复合链接函数中，我们现在可以对收集到的节点链接函数进行调用：

```js
function compileNodes($compileNodes) {
  // var linkFns = [];
  // _.forEach($compileNodes, function(node, i) {
  //   var attrs = new Attributes($(node));
  //   var directives = collectDirectives(node, attrs);
  //   var nodeLinkFn;
  //   if (directives.length) {
  //     nodeLinkFn = applyDirectivesToNode(directives, node, attrs);
  //   }
  //   if ((!nodeLinkFn || !nodeLinkFn.terminal) &&
  //     node.childNodes && node.childNodes.length) {
  //     compileNodes(node.childNodes);
  //   }
  //   if (nodeLinkFn) {
  //     linkFns.push({
  //       nodeLinkFn: nodeLinkFn,
  //       idx: i
  //     });
  //   }
  // });

  // function compositeLinkFn(scope, linkNodes) {
    _.forEach(linkFns, function(linkFn) {
      linkFn.nodeLinkFn(scope, linkNodes[linkFn.idx]);
    });
  // }
  // return compositeLinkFn;
}
```

我们希望编译节点和链接节点是一一对应的，因为我们认为它们索引应该是一致的。这个假设开始并不能确保实现，而现在我们已经可以支持。

最后，我们会对单独节点的链接函数进行处理，也就是到了我们可以对指令本身进行链接的阶段了。我们需要收集指令链接函数，这可以通过调用各个指令的`compile`函数获取到：

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  var linkFns = [];
  // _.forEach(directives, function(directive) {
  //   if (directive.$$start) {
  //     $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  //   }
  //   if (directive.priority < terminalPriority) {
  //     return false;
  //   }
  //   if (directive.compile) {
      var linkFn = directive.compile($compileNode, attrs);
      if (linkFn) {
        linkFns.push(linkFn);
      }
  //   }
  //   if (directive.terminal) {
  //     terminal = true;
  //     terminalPriority = directive.priority;
  //   }
  // });
  // return terminal;
}
```

现在就可以构建节点的链接函数，并进行返回了。这个函数会调用指令的链接函数。我们也需要把`terminal`标识变成节点链接函数的一个属性，这样`compileNodes`函数才能使用这个标识：

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // var linkFns = [];
  // _.forEach(directives, function(directive) {
  //   if (directive.$$start) {
  //     $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  //   }
  //   if (directive.priority < terminalPriority) {
  //     return false;
  //   }
  //   if (directive.compile) {
  //     var linkFn = directive.compile($compileNode, attrs);
  //     if (linkFn) {
  //       linkFns.push(linkFn);
  //     }
  //   }
  //   if (directive.terminal) {
  //     terminal = true;
  //     terminalPriority = directive.priority;
  //   }
  // });

  function nodeLinkFn(scope, linkNode) {
    _.forEach(linkFns, function(linkFn) {
      var $element = $(linkNode);
      linkFn(scope, $element, attrs);
    });
  }
  nodeLinkFn.terminal = terminal;
  return nodeLinkFn;
}
```

现在，我们的测试用例终于通过了，也就是我们可以进行一些简单的链接工作了。就如我们看到的那样，这实际上包含几个步骤，而这几个步骤都有各自具体的任务：

- 公共链接函数是用于对我们要编译的整棵 DOM 树进行链接的
- 复合链接函数会对一个集合中的所有节点进行链接
- 节点链接函数会对一个节点上的所有指令进行链接
- 指令链接函数会对单个指令进行链接

应用开发者可以使用的是第一个和最后一个链接方法。而其中的两个步骤都是`compile.js`中会进行的内部过程。

![compile and link](/assets/18-directive-linking-scopes/compile-and-link.png)