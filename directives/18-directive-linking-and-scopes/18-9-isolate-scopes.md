### 独立作用域（Isolate Scope）

回忆一下我们在第二章中看到的做作用域继承的两种方法：原型式继承，还有非原型式继承，即独立作用域。我们已经看到第一种方法是如何与指令进行绑定的。下面我们来看看第二种继承方法是如何与指令进行集成的。

我们已经知道了，独立作用域是指虽然属于作用域继承层次的一员，但却不继承父作用域属性的作用域。它们跟其它作用域一样，参与了事件冒泡和 digest 循环，但我们无法使用它在父子作用域之间传递任意数据。当你对指令使用一个独立作用域，这会让指令更加模块化，因为你可以确保你的指令或多或少已经与周围环境独立开来了。

然而实际上独立作用域其实也不是与它们的上下文完全脱离开来。通过使用独立作用域绑定的方法，指令系统允许我们在周围的环境中绑定一些属性到独立作用域。这种绑定的方式与普通的作用域继承相比最大的区别是，我们必须显式地把所有要传递到独立作用域的东西定义为一个独立作用域绑定，而普通的作用域继承会把原型链上所有的父对象属性特性都继承下来，无论你是否需要它们。

在开始讲解绑定之前，我们先来做些基础工作。我们会通过用一个对象作为指令定义对象`scope`属性的值来请求生成一个独立作用域。而对应的指令作用域会成为当前作用域上下文的一个子作用域，但并不是通过当前作用域上文的原型链来生成继承关系：

_test/compile\_spec.js_

```js
it('creates an isolate scope when requested', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {},
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(givenScope.$parent).toBe($rootScope);
    expect(Object.getPrototypeOf(givenScope)).not.toBe($rootScope);
  });
});
```

独立作用域指令还有一个重要的特点，也是与普通作用域指令的另一个区别点，当一个指令使用的是独立作用域，这个作用域并不会传递给同元素上的其它指令。也就是说，独立作用域的“独立”是针对指令而不是元素节点：

_test/compile\_spec.js_

```js
it('does not share isolate scope with other directives', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: {}
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
    expect(givenScope).toBe($rootScope);
  });
});
```

正如我们在这个单元测试看到的，当元素上有两个指令，其中一个指令用的是独立作用域，另一个指令会依然使用从上下文获得的作用域：

同样的规则，对元素的子节点同样有效。传入子节点的不会是独立作用域：

_test/compile\_spec.js_

```js
it('does not use isolate scope on child elements', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: {}
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
    var el = $('<div my-directive><div my-other-directive></div></div>');
    $compile(el)($rootScope);
    expect(givenScope).toBe($rootScope);
  });
});
```

> 对于最后这条规则来说其实也会发生例外情况：有时元素的子节点也会共享一个独立作用域。比如，当子节点是通过独立作用域指令的模板创建出来的时候就会发生例外情况，在后面的章节我们会看到。

配置好对独立作用域的基本测试组合后，我们可以开始编写我们需要的代码了。

在`applyDirectivesToNode`中，我们需要判断指令需要的是否是一个独立作用域，并在调用`addLinkFns`把判断结果作为参数传入：

_src/compile.js_

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // var preLinkFns = [],
  //   postLinkFns = [];
  var newScopeDirective, newIsolateScopeDirective;

  function addLinkFns(preLinkFn, postLinkFn, attrStart, attrEnd, isolateScope) {
    // ...
  }

  // _.forEach(directives, function(directive) {
  //   if (directive.$$start) {
  //     $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  //   }
  //   if (directive.priority < terminalPriority) {
  //     return false;
  //   }
  //   if (directive.scope) {
      if (_.isObject(directive.scope)) {
        newIsolateScopeDirective = directive;
      } else {
        // newScopeDirective = newScopeDirective || directive;
      }
    // }
    // if (directive.compile) {
    //   var linkFn = directive.compile($compileNode, attrs);
      var isolateScope = (directive === newIsolateScopeDirective);
      // var attrStart = directive.$$start;
      // var attrEnd = directive.$$end;
      // if (_.isFunction(linkFn)) {
        addLinkFns(null, linkFn, attrStart, attrEnd, isolateScope);
      // } else if (linkFn) {
        addLinkFns(linkFn.pre, linkFn.post, attrStart, attrEnd, isolateScope);
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

在`addLinkFns`函数中，我们就直接把`isolateScope`标识赋值给链接前和链接后函数：

```js
function addLinkFns(preLinkFn, postLinkFn, attrStart, attrEnd, isolateScope) {
  // if (preLinkFn) {
  //   if (attrStart) {
  //     preLinkFn = groupElementsLinkFnWrapper(preLinkFn, attrStart, attrEnd);
  //   }
    preLinkFn.isolateScope = isolateScope;
  //   preLinkFns.push(preLinkFn);
  // }
  // if (postLinkFn) {
  //   if (attrStart) {
  //     postLinkFn = groupElementsLinkFnWrapper(postLinkFn, attrStart, attrEnd);
  //   }
    postLinkFn.isolateScope = isolateScope;
  //   postLinkFns.push(postLinkFn);
  // }
}
```

然后，在节点链接函数中，我们就需要按照需要创建真正的独立作用域：

_src/compile.js_

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  // var $element = $(linkNode);

  var isolateScope;
  if (newIsolateScopeDirective) {
    isolateScope = scope.$new(true);
  }

  // _.forEach(preLinkFns, function(linkFn) {
  //   linkFn(scope, $element, attrs);
  // });

  // if (childLinkFn) {
  //   childLinkFn(scope, linkNode.childNodes);
  // }
  // _.forEachRight(postLinkFns, function(linkFn) {
  //   linkFn(scope, $element, attrs);
  // });
}
```

然后如果我们发现链接函数带上了`isolateScope`标识，我们就会把独立作用域传递给这个链接函数。这意味着这个链接函数来自一个独立作用域指令，它会接收一个独立作用域，而其它指令会接收默认的上下文作用域。另外，子链接函数永远不会接收到一个独立作用域：

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  // var $element = $(linkNode);

  // var isolateScope;
  // if (newIsolateScopeDirective) {
  //   isolateScope = scope.$new(true);
  // }

  // _.forEach(preLinkFns, function(linkFn) {
    linkFn(linkFn.isolateScope ? isolateScope : scope, $element, attrs);
  // });
  // if (childLinkFn) {
  //   childLinkFn(scope, linkNode.childNodes);
  // }
  // _.forEachRight(postLinkFns, function(linkFn) {
    linkFn(linkFn.isolateScope ? isolateScope : scope, $element, attrs);
  // });
}
```

正如我们看到的，一个指令的独立作用域并不会被同元素上的其它指令或子元素所共享。进一步来说，其实我们只允许一个元素上有一个指令可以拥有独立作用域。如果我们尝试对一个元素注册多个独立作用域指令，那在编译阶段就会抛出错误：

```js
it('does not allow two isolate scope directives on an element', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: {}
      };
    },
    myOtherDirective: function() {
      return {
        scope: {}
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    expect(function() {
      $compile(el);
    }).toThrow();
  });
})
```

事实上，如果元素上有一个独立作用域指令，那该元素上的其它指令也不能注册一个非独立的继承作用域：

```js
it('does not allow both isolate and inherited scopes on an element', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        scope: {}
      };
    },
    myOtherDirective: function() {
      return {
        scope: true
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    expect(function() {
      $compile(el);
    }).toThrow();
  });
});
```

我们会在`applyDirectivesToNode`来进行条件判断，在这里有两条判断规则：

1. 遇上了一个独立作用域，而之前已经遇到了另一个独立作用域或者继承作用域。
2. 遇上了一个继承作用域，而之前已经遇到了一个独立作用域了。

对于这两种情况，我们会抛出一个提示信息，而这个信息是很多 Angular 应用开发者之前已经看到过的：

_src/compile.js_

```js
_.forEach(directives, function(directive) {
  // if (directive.$$start) {
  //   $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  // }
  // if (directive.priority < terminalPriority) {
  //   return false;
  // }
  // if (directive.scope) {
  //   if (_.isObject(directive.scope)) {
      if (newIsolateScopeDirective || newScopeDirective) {
        throw 'Multiple directives asking for new/inherited scope';
      }
    //   newIsolateScopeDirective = directive;
    // } else {
      if (newIsolateScopeDirective) {
        throw 'Multiple directives asking for new/inherited scope';
      }
  //     newScopeDirective = newScopeDirective || directive;
  //   }
  // }
  // ...
});
```

最后，如果一个元素应用了独立作用域指令，它会被附加上一个`ng-isolate-scope`的样式类（也就意味着它不会被附加上`ng-scope`样式类），而其作用域对象将会成为一个名为`$isolateScope`的 jQuery 数据：

_test/compile.spec.js_

```js
it('adds class and data for element with isolated scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {},
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect(el.hasClass('ng-isolate-scope')).toBe(true);
    expect(el.hasClass('ng-scope')).toBe(false);
    expect(el.data('$isolateScope')).toBe(givenScope);
  });
});
```

这两步会在创建独立作用域后的节点链接函数中完成：

_src/compile.js_

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  var $element = $(linkNode);

  var isolateScope;
  if (newIsolateScopeDirective) {
    isolateScope = scope.$new(true);
    $element.addClass('ng-isolate-scope');
    $element.data('$isolateScope', isolateScope);
  }

  // ...

}
```



