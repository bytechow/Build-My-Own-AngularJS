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