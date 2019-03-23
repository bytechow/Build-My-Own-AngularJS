### 终止编译（Terminating Compilation）

通常情况下，当我们让 Angular 去编译 DOM 元素，那在这个 DOM 元素被编译后，其子树也会紧随其后被编译。这是由我们本章实现的编译器特性决定的。但我们还可能遇到不需要所有指令都被编译的情况。其中一种情况就是我们需要在 DOM 中应用的指令是一个终止指令（terminal directive）。

我们可以通过在指令的定义对象中设置`terminal`属性为`true`来把指令标记为终止指令。设置指令为终止状态后，当这个指令被编译时，它的编译过程会被终止，它子元素上的指令也不会被编译。

终止指令常用在我们希望延迟某个指令的编译动作的情况。举例来说，Angular 内建的`ng-if`就是一个终止指令，使用这个指令后，它所在的 DOM 子树也会全部终止掉编译。当这个指令内的条件表达式的计算结果值为真值（truthy），它会再启动编译。这里使用到了我们在后面章节实现的 transclusion（嵌入包含）特性。

```html
<div ng-if="condition">
  <!-- Contents compiled later when condition is true -->
  <div some-other-directive></div>
</div>
```

如前所述，当在一个 HTML 节点上加上了终止标识，其它比这个终止指令优先级低的指令都不会被编译：

_test/compile_spec.js_

```js
it('stops compiling at a terminal directive', function() {
  var compilations = [];
  var myModule = window.angular.module('myModule', []);
  myModule.directive('firstDirective', function() {
    return {
      priority: 1,
      terminal: true,
      compile: function(element) {
        compilations.push('first');
      }
    };
  });
  myModule.directive('secondDirective', function() {
    return {
      priority: 0,
      compile: function(element) {
        compilations.push('second');
      }
    };
  });
  var injector = createInjector(['ng', 'myModule']);
  injector.invoke(function($compile) {
    var el = $('<div first-directive second-directive></div>');
    $compile(el);
    expect(compilations).toEqual(['first']);
  });
});
```

然而，如果其它指令与终止指令的优先级一致，它们还是会被编译：

```js
it('still compiles directives with same priority after terminal', function() {
  var compilations = [];
  var myModule = window.angular.module('myModule', []);
  myModule.directive('firstDirective', function() {
    return {
      priority: 1,
      terminal: true,
      compile: function(element) {
        compilations.push('first');
      }
    };
  });
  myModule.directive('secondDirective', function() {
    return {
      priority: 1,
      compile: function(element) {
        compilations.push('second');
      }
    };
  });
  var injector = createInjector(['ng', 'myModule']);
  injector.invoke(function($compile) {
    var el = $('<div first-directive second-directive></div>');
    $compile(el);
    expect(compilations).toEqual(['first', 'second']);
  });
});
```

`applyDirectivesToNode`是实际执行实际编译指令的地方，在这里，我们需要记录终止指令的优先级。我们会使用 JavaScript 中的一个极小值（-Number.MAX_VALUE）作为终止指令的默认初始值，如果遇到一个终止指令我们会用该指令的优先级更新终止指令的默认优先级：

_src/compile.js_

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  var terminalPriority = -Number.MAX_VALUE;
  // _.forEach(directives, function(directive) {
  //   if (directive.compile) {
  //     directive.compile($compileNode, attrs);
  //   }
    if (directive.terminal) {
      terminalPriority = directive.priority;
    }
  // });
}
```

如果我们在遍历编译的过程中遇到由指令的优先级是小于当前终止指令的优先级的，就用`return false`的方式提前退出当前循环，这就能保证不对当前指令进行实际的编译了：

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // _.forEach(directives, function(directive) {
    if (directive.priority < terminalPriority) {
      return false;
    }
  //   if (directive.compile) {
  //     directive.compile($compileNode, attrs);
  //   }
  //   if (directive.terminal) {
  //     terminalPriority = directive.priority;
  //   }
  // });
}
```

当一个元素上被应用了终止指令后，我们还需要保证它子节点上的指令也不会被编译，但我们目前还是会对这种情况下的字节点指令编译，我们应该对其进行禁止：

_test/compile_spec.js_

```js
it('stops child compilation after a terminal directive', function() {
  var compilations = [];
  var myModule = window.angular.module('myModule', []);
  myModule.directive('parentDirective', function() {
    return {
      terminal: true,
      compile: function(element) {
        compilations.push('parent');
      }
    };
  });
  myModule.directive('childDirective', function() {
    return {
      compile: function(element) {
        compilations.push('child');
      }
    };
  });
  var injector = createInjector(['ng', 'myModule']);
  injector.invoke(function($compile) {
    var el = $('<div parent-directive><div child-directive></div></div>');
    $compile(el);
    expect(compilations).toEqual(['parent']);
  });
});
```

现在，我们还需要在`applyDirectivesToNode`方法的最后返回一个标识，这个标识代表在这个 DOM 节点上是否有应用过终止指令：

_src/compile.js_

```js
function applyDirectivesToNode(directives, compileNode) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  var terminal = false;
  // _.forEach(directives, function(directive) {
  //   if (directive.priority < terminalPriority) {
  //     return false;
  //   }
  //   if (directive.compile) {
  //     directive.compile($compileNode);
  //   }
  //   if (directive.terminal) {
      terminal = true;
  //     terminalPriority = directive.priority;
  //   }
  // });
  return terminal;
}
```

有了这个标识，我们就可以在`compileNodes`方法中就可以获知到当前节点是否包含终止指令，然后我们自然就可以以此判断是否要继续对该节点下的子节点进行编译：

```js
function compileNodes($compileNodes) {
  _.forEach($compileNodes, function(node) {
    var directives = collectDirectives(node);
    var terminal = applyDirectivesToNode(directives, node);
    if (!terminal && node.childNodes && node.childNodes.length) {
      compileNodes(node.childNodes);
    }
  });
}
```

之后我们会对管理终止标识进行更深入的探究，但目前来说我们算是把终止指令的主要功能都实现了。