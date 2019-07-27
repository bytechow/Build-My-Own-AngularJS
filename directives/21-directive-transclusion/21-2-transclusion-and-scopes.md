### Transclusion 与作用域（Transclusion And Scopes）

如果要在经过 transclusion 的内容里使用任意指令，它们需要进行链接。其实我们已经是那样做，因为我们目前是用公共链接函数作为 transclusion 函数。但这个链接过程并没有加入作用域，只要我们在经过 transclusion 的内容里使用作用域就能清楚地看到这个问题：

_test/compile_spec.js_

```js
it('makes scope available to link functions inside', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true,
        link: function(scope, element, attrs, ctrl, transclude) {
          element.append(transclude());
        }
      };
    },
    myInnerDirective: function() {
      return {
        link: function(scope, element) {
          element.html(scope.anAttr);
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div my-inner-directive></div></div>');
    $rootScope.anAttr = 'Hello from root';
    $compile(el)($rootScope);
    expect(el.fnd('> [my-inner-directive]').html()).toBe('Hello from root');
  });
});
```

我们可以通过提前绑定（pre-binding）transclusion 函数到一个作用域来修复这个问题。实际上，我们把带作用域参数的 transclusion 函数调用包裹在另一个函数中，并把调用结果变成包裹函数的返回值。这也意味着指令创建者不需要自己提供作用域，我们会在指令系统中完成这个任务。

_src/compile.js_

```js
function boundTranscludeFn() {
  return childTranscludeFn(scope);
}

_.forEach(preLinkFns, function(linkFn) {
  linkFn(
  //   linkFn.isolateScope ? isolateScope : scope,
  //   $element,
  //   attrs,
  //   linkFn.require && getControllers(linkFn.require, $element),
    boundTranscludeFn
  );
});
// if (childLinkFn) {
//   var scopeToChild = scope;
//   if (newIsolateScopeDirective && newIsolateScopeDirective.template) {
//     scopeToChild = isolateScope;
//   }
//   childLinkFn(scopeToChild, linkNode.childNodes);
// }
_.forEachRight(postLinkFns, function(linkFn) {
  linkFn(
    // linkFn.isolateScope ? isolateScope : scope,
    // $element,
    // attrs,
    // linkFn.require && getControllers(linkFn.require, $element),
    boundTranscludeFn
  );
})
```

> 如果有读者熟悉函数式编程的话，就会认出这其实是[偏函数应用](https://en.wikipedia.org/wiki/Partial_application)。其实我们可以用 Lodash 进行简化这个表达式，`var boundTranscludeFn = _.partial(childTranscludeFn, scope)`，但我们之后会在 boundTranscludeFn 中做其他事，所以就用原生的定义方式就好了。

我们现在绑定到 transclusion 函数的作用域还不是十分恰当。要 transclude 的内容应该链接到当初定义它们的地方所对应的作用域，而现在默认使用一个之前用过的作用域进行链接。例如，如果 transclusion 指令创建了一个继承作用域，那经过 transclude 的内容应该对这个作用域一无所知才是。当 transclusion 指令遮蔽了父作用域上的一个属性，我们就能发现问题。经过 transclude 的内容的作用域也应该仍然能访问到父作用域上的属性值，但实际上并不是这样的：

_test/compile_spec.js_

```js
t('does not use the inherited scope of the directive', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true,
        scope: true,
        link: function(scope, element, attrs, ctrl, transclude) {
          scope.anAttr = 'Shadowed attribute';
          element.append(transclude());
        }
      };
    },
    myInnerDirective: function() {
      return {
        link: function(scope, element) {
          element.html(scope.anAttr);
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div my-inner-directive></div></div>');
    $rootScope.anAttr = 'Hello from root';
    $compile(el)($rootScope);
    expect(el.fnd('> [my-inner-directive]').html()).toBe('Hello from root');
  });
});
```

这对我们已绑定的 transclusion 函数出了一个难题，因为这个函数是在只能访问到继承作用域的节点链接函数中创建的。为了能准确判断用哪个作用域进行 transclusion，我们需要在_组合链接函数_中进行创建。

但首先，当一个元素节点有 transclusion 指令时，我们需要让组合链接函数知道。我们可以把两个新的跟踪变量传给节点链接函数作为属性：

_src/compile.js_

```js
function applyDirectivesToNode(
  directives, compileNode, attrs, previousCompileContext) {
  
  // ...
  
  // nodeLinkFn.terminal = terminal;
  // nodeLinkFn.scope = newScopeDirective && newScopeDirective.scope;
  nodeLinkFn.transcludeOnThisElement = hasTranscludeDirective;
  nodeLinkFn.transclude = childTranscludeFn;
  
  return nodeLinkFn;
}
```

在组合链接函数里面，我们现在可以设置包裹的 transclusion 函数了。首先，我们需要改变目前的代码，为节点创建一个继承作用域，这样就不会遮蔽父作用域上的变量，而是会使用一个独立的变量：

_src/compile.js_

```js
_.forEach(linkFns, function(linkFn) {
  // var node = stableNodeList[linkFn.idx];
  if (linkFn.nodeLinkFn) {
    var childScope;
    if (linkFn.nodeLinkFn.scope) {
      childScope = scope.$new();
      $(node).data('$scope', childScope);
    } else {
      childScope = scope;
    }
    linkFn.nodeLinkFn(
      // linkFn.childLinkFn,
      childScope,
      // node,
      // boundTranscludeFn
    );
  } else {
    linkFn.childLinkFn(
      scope,
      node.childNodes
    );
  }
});
```

现在，如果节点上有 transclusion 指令（使得`transcludeOnThisElement`变成`true`）,我们就创建一个包裹的 transclusion 函数。它会以当前上下文作为参数调用原始的 transclusion 函数（也就是当前节点链接函数的`transclude`属性）。之后，我们会将包裹的 transclusion 函数作为参数传给节点链接函数：

```js
_.forEach(linkFns, function(linkFn) {
  var node = stableNodeList[linkFn.idx];
  if (linkFn.nodeLinkFn) {
    // var childScope;
    // if (linkFn.nodeLinkFn.scope) {
    //   childScope = scope.$new();
    //   $(node).data('$scope', childScope);
    // } else {
    //   childScope = scope;
    // }

    var boundTranscludeFn;
    if (linkFn.nodeLinkFn.transcludeOnThisElement) {
      boundTranscludeFn = function() {
        return linkFn.nodeLinkFn.transclude(scope);
      };
    }
    
    linkFn.nodeLinkFn(
      // linkFn.childLinkFn,
      // childScope,
      // node,
      boundTranscludeFn
    );
  } else {
    linkFn.childLinkFn(
      scope,
      node.childNodes
    );
  }
});
```

现在节点链接函数有一个新的参数需要接收。加入它的同时，我们需要把`function boundTranscludeFn`的声明从节点链接函数中移除掉，毕竟我们已经从参数接收了一个包裹的 transclusion 函数，也就没必要再声明一个了。

```js
function nodeLinkFn(childLinkFn, scope, linkNode, boundTranscludeFn) {
  
  // ...
  
}
```

我们现在完成的这个版本的包裹的 transclusion 函数，是在 transclusion 指令的外部进行作用域的绑定的。经过 transclude 的内容现在可以访问到需要的 Scope 内容。

但这个作用域还依然不完全是我们需要的作用域。虽然它确实持有数据，还有我们需要的原型链继承，还有一个与作用域生命周期相关的问题：当 transclusion 指令的作用域被销毁的时候，我们希望 transclusion 内容中的监视器和事件监听也应该被销毁。目前还没实现这个，因为我们用于 transclusion 的是上下文作用域，这个作用域可能会在 transclusion 指令销毁之后仍然存在一段时间。

_test/compile_spec.js_

```js
it('stops watching when transcluding directive is destroyed', function() {
  var watchSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true,
        scope: true,
        link: function(scope, element, attrs, ctrl, transclude) {
          element.append(transclude());
          scope.$on('destroyNow', function() {
            scope.$destroy();
          });
        }
      };
    },
    myInnerDirective: function() {
      return {
        link: function(scope) {
          scope.$watch(watchSpy);
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div my-inner-directive></div></div>');
    $compile(el)($rootScope);

    $rootScope.$apply();
    expect(watchSpy.calls.count()).toBe(2);
    
    $rootScope.$apply();
    expect(watchSpy.calls.count()).toBe(3);
    
    $rootScope.$broadcast('destroyNow');
    $rootScope.$apply();
    expect(watchSpy.calls.count()).toBe(3);
  });
});
```

这里我们会在 transclusion 里面注册一个观察者。我们期望它会在 transclusion 指令接收后中止运作，但现在却不是这样的。

从这里，我们也能看到 transclusion 作用域和其他作用域的区别是什么：它们从中获取数据的父作用域跟要决定它们是否要被销毁的父作用域是不同的。也就是说，它们（transclusion 作用域）需要两个父作用域。

回顾第二章，我们确实实现了利用传入的 Scope 对象实现两个父作用域的功能。当我们调用`scope.$new()`时，我们可以传入可选的第二个参数：将会成为新作用域的`$parent`的 Scope 对象。这个作用域将会决定新作用域在什么时候会被销毁，而 JavaScript 原型（也就是所有的数据）会依然设置为调用`$new`方法的 Scope 对象。

现在我们就可以利用这个特性了：我们需要创建一个特殊的 transclusion 作用域，这个作用域的原型是上下文作用域，而`$parent`会被设置为 transclusion 指令的作用域。

后面说的这个作用域会在节点链接函数中使用，它会创建第二层的绑定数据，并被带到 transclusion 函数中，这个函数就是绑定了作用域的 transclusion 函数：

_src/compile.js_

```js
function scopeBoundTranscludeFn() {
  return boundTranscludeFn(scope);
}
_.forEach(preLinkFns, function(linkFn) {
  linkFn(
    // linkFn.isolateScope ? isolateScope : scope,
    // $element,
    // attrs,
    // linkFn.require && getControllers(linkFn.require, $element),
    scopeBoundTranscludeFn
  );
});
// if (childLinkFn) {
//   var scopeToChild = scope;
//   if (newIsolateScopeDirective && newIsolateScopeDirective.template) {
//     scopeToChild = isolateScope;
//   }
//   childLinkFn(scopeToChild, linkNode.childNodes);
// }
_.forEachRight(postLinkFns, function(linkFn) {
  linkFn(
    // linkFn.isolateScope ? isolateScope : scope,
    // $element,
    // attrs,
    // linkFn.require && getControllers(linkFn.require, $element),
    scopeBoundTranscludeFn
  );
});
```

当你在指令中接收到一个 transclusion 函数时，实际上接收到的事：被 transclude 内容的原生链接函数，被包裹到两个不同的绑定函数中。

![](/assets/when-receive-a-transclusion-function.png)