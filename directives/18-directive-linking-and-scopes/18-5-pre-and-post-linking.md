### Pre- And Post-Linking

可能会因为元素子节点比元素本身更先被链接这个事实感到奇怪。对此，有一个比较好的解释，其实链接函数是有两种类型的，目前我们只实现了他们其中的一个而已。实际上，我们会把链接函数分为链接前函数（prelink）和链接后函数（postlink）两种。这两种函数的区别就在于它们的调用顺序。链接前函数会在子节点进行链接之前，而链接后函数会在子节点进行链接之后执行。

我们之前调用的链接函数实际上是链接后函数，如果我们不进行指定，那链接后函数会成为默认调用的链接函数。另一个更直观地看到我们目前已实现的方法是在指令定义对象里加入一个内嵌在 link 对象属性下的 `post` 属性：

_test/compile_spec.js_

```js
it('supports link function objects', function() {
  var linked;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      link: {
        post: function(scope, element, attrs) {
          linked = true;
        }
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div><div my-directive></div></div>');
    $compile(el)($rootScope);
    expect(linked).toBe(true);
  });
});
```

当我们在编译一个节点时，我们需要先检查下我们是否有直接指定的链接函数，还是拥有一个链接函数对象，里面会有一个`post`属性：

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
      if (_.isFunction(linkFn)) {
        // linkFns.push(linkFn);
      } else if (linkFn) {
        linkFns.push(linkFn.post);
      }
  //   }
  //   if (directive.terminal) {
  //     terminal = true;
  //     terminalPriority = directive.priority;
  //   }
  // });

  // ...
  
}
```

我们使用对象来指定链接函数就是希望可以同时支持链接前和链接后函数。下面我们会构建一个两层的节点结构，并检查链接函数的执行顺序：

_test/compile_spec.js_

```js
it('supports prelinking and postlinking', function() {
  var linkings = [];
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      link: {
        pre: function(scope, element) {
          linkings.push(['pre', element[0]]);
        },
        post: function(scope, element) {
          linkings.push(['post', element[0]]);
        }
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive><div my-directive></div></div>');
    $compile(el)($rootScope);
    expect(linkings.length).toBe(4);
    expect(linkings[0]).toEqual(['pre', el[0]]);
    expect(linkings[1]).toEqual(['pre', el[0].frstChild]);
    expect(linkings[2]).toEqual(['post', el[0].frstChild]);
    expect(linkings[3]).toEqual(['post', el[0]]);
  });
});
```

我们需要确保链接函数是按以下顺序执行的：

1. 父元素的链接前函数
2. 子元素的链接前函数
3. 子元素的链接后函数
4. 父元素的链接后函数

当前单元测试会失败，因为链接前函数根本没有被调用。

我们会对`applyDirectivesToNode`方法进行改变，这样它就可以分别收集链接前函数和链接后函数，分别用`preLinkFns`和`postLinkFns`存储：

_src/compile.js_

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  var preLinkFns = [],
    postLinkFns = [];
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
        postLinkFns.push(linkFn);
      // } else if (linkFn) {
        if (linkFn.pre) {
          preLinkFns.push(linkFn.pre);
        }
        if (linkFn.post) {
          postLinkFns.push(linkFn.post);
        }
    //   }
    // }
    // if (directive.terminal) {
    //   terminal = true;
    //   terminalPriority = directive.priority;
    // }
  // });
}
```

然后我们需要对节点链接函数进行修改，以便对调用顺序进行支持：首先我们需要调用链接前函数，然后调用子节点的链接函数，最后调用链接后函数：

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  var $element = $(linkNode);
  _.forEach(preLinkFns, function(linkFn) {
    linkFn(scope, $element, attrs);
  });
  // if (childLinkFn) {
  //   childLinkFn(scope, linkNode.childNodes);
  // }
  _.forEach(postLinkFns, function(linkFn) {
    // linkFn(scope, $element, attrs);
  });
}
```

在之前的章节中，我们选择传递子节点的链接函数给节点链接函数而不是直接对其进行调用。现在我们就明白其中的原因了：这能够让我们有机会在调用子节点的链接函数之前调用链接前函数。

![](/assets/18-directive-linking-and-scopes/pre-and-post-link.png)

链接前函数和链接后函数还有一个区别，这个跟它们在一个元素中调用的顺序有关：链接前函数是按照指令优先级的顺序进行调用的，但链接后函数实际上会按指令优先级的相反顺序进行调用。这是关于链接后函数的一个通用规则，无论对于跨元素还是单元素来说都一样：它们的调用顺序会与编译顺序正好相反。

我们目前的实现代码仍然会按照优先级顺序对两种链接函数进行调用，这是由于我们是在编译时对它们进行按序的收集。下面这个测试将不会通过：

_test/compile_spec.js_

```js
it('reverses priority for postlink functions', function() {
  var linkings = [];
  var injector = makeInjectorWithDirectives({
    firstDirective: function() {
      return {
        priority: 2,
        link: {
          pre: function(scope, element) {
            linkings.push('first-pre');
          },
          post: function(scope, element) {
            linkings.push('first-post');
          }
        }
      };
    },
    secondDirective: function() {
      return {
        priority: 1,
        link: {
          pre: function(scope, element) {
            linkings.push('second-pre');
          },
          post: function(scope, element) {
            linkings.push('second-post');
          }
        }
      };
    },
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div first-directive second-directive></div>');
    $compile(el)($rootScope);
    expect(linkings).toEqual([
      'first-pre',
      'second-pre',
      'second-post',
      'first-post'
    ]);
  });
});
```

我们可以通过修改对链接后函数的遍历方法，让遍历顺序变成由右向左：

_src/compile.js_

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  // var $element = $(linkNode);
  // _.forEach(preLinkFns, function(linkFn) {
  //   linkFn(scope, $element, attrs);
  // });
  // if (childLinkFn) {
  //   childLinkFn(scope, linkNode.childNodes);
  // }
  _.forEachRight(postLinkFns, function(linkFn) {
  //   linkFn(scope, $element, attrs);
  // });
}
```