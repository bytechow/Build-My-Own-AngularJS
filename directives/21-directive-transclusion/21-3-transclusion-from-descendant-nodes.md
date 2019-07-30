### 在后代节点中进行 transclusion（Transclusion from Descendant Nodes）

正如我们所看到的，当你有一个带有`transclude: true`属性的指令时，它的链接函数会在第五个参数处接收到一个 transclusion 函数，这让指令能够访问到经过 transclude 的内容。事实上，这时候元素上的所有指令都能接受到这第五个参数，因为我们会把这个 transclusion 函数传递给现有的所有 pre-link 和 post-link 函数。

其实，还有更多的指令可以访问到这个 transclusion 函数：只要当前元素或它的后代元素是有 transclusion 的，都能访问到这第五个参数。这就意味着，我们可以在 transclusion 指令的模板内部元素摆放经过 transclude 的内容，就像我们下面这样：

_test/compile\_spec.js_

```js
it('makes contents available to child elements', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true,
        template: '<div in-template></div>'
      };
    },
    inTemplate: function() {
      return {
        link: function(scope, element, attrs, ctrl, transcludeFn) {
          element.append(transcludeFn());
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div in-transclude></div></div>');
    $compile(el)($rootScope);
    expect(el.fnd('> [in-template] > [in-transclude]').length).toBe(1);
  });
});
```

现在这个单元测试没通过，原因是`inTemplate`指令并没有接收到一个 transclusion 函数，所以它现在调用的只是一个`undefined`。

在节点链接函数，正如我们调用子节点的链接函数一样，我们也应该把`boundTranscludeFn`传入到子节点中，让子节点也可以访问到：

_src/compile.js_

```js
// _.forEach(preLinkFns, function(linkFn) {
//   linkFn(
//     linkFn.isolateScope ? isolateScope : scope,
//     $element,
//     attrs,
//     linkFn.require && getControllers(linkFn.require, $element),
//     scopeBoundTranscludeFn
//   );
// });
if (childLinkFn) {
  // var scopeToChild = scope;
  // if (newIsolateScopeDirective && newIsolateScopeDirective.template) {
  //   scopeToChild = isolateScope;
  // }
  childLinkFn(scopeToChild, linkNode.childNodes, boundTranscludeFn);
}
// _.forEachRight(postLinkFns, function(linkFn) {
//   linkFn(
//     linkFn.isolateScope ? isolateScope : scope,
//     $element,
//     attrs,
//     linkFn.require && getControllers(linkFn.require, $element),
//     scopeBoundTranscludeFn
//   );
// });
```

注意，我们没有传递绑定了 scope 的 transclusion 函数，而只是内部的`boundTranscludeFn`函数。而实际上会由子节点构建自己的绑定 scope 的 transclusion 函数。

被调用的链接函数就是子节点的组合链接函数。这个函数还没准备好接收第三个参数。我们会增加这个参数，并称它为`parentBoundTranscludeFn`，因为它就是父节点的一个绑定的 transclusion 函数：

_src/compile.js_

```js
function compositeLinkFn(scope, linkNodes, parentBoundTranscludeFn) {
  // ...
}
```

现在，在这个函数里面的对链接函数的遍历中，我们会使用这个`parentBoundTranscludeFn`作为绑定的 transclude 函数，但前提是这个子节点自己本身没有做任何的 transclusion：

```js
var boundTranscludeFn;
if (linkFn.nodeLinkFn.transcludeOnThisElement) {
  boundTranscludeFn = function(transcludedScope, containingScope) {
    if (!transcludedScope) {
      transcludedScope = scope.$new(false, containingScope);
    }
    return linkFn.nodeLinkFn.transclude(transcludedScope);
  };
} else if (parentBoundTranscludeFn) {
  boundTranscludeFn = parentBoundTranscludeFn;
}
```

这样就能够通过上面的单元测试了：现在我们可以把祖先节点上的 transclude 内容放到子节点上了。最终 transclusion 作用域的生命周期依然由进行 transclusion 的位置来决定。

DOM 上的所有元素并不都拥有指令，我们也要能把 transclusion 函数传递给它们才行。在这个测试中，我们在要进行 transclusion 的指令模板中会有一个不带指令的`div`元素，这会令`in-template`指令无法接收到这个 transclude 函数：

_test/compile\_spec.js_

```js
it('makes contents available to indirect child elements', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true,
        template: '<div><div in-template></div></div>'
      };
    },
    inTemplate: function() {
      return {
        link: function(scope, element, attrs, ctrl, transcludeFn) {
          element.append(transcludeFn());
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div in-transclude></div></div>');

    $compile(el)($rootScope);

    expect(el.fnd('> div > [in-template] > [in-transclude]').length).toBe(1);
  });
});
```

我们会再次遇上之前把`undefined`当作 transclusion 函数进行调用的错误。

这个我们可以通过在组合链接函数中不断往前传递从父节点传入的绑定 transclusion 函数来解决，也就是对那些没有链接函数的节点也传入：

_src/compile.js_

```js
if (linkFn.nodeLinkFn) {
  // ...
} else {
  linkFn.childLinkFn(
    // scope,
    // node.childNodes,
    parentBoundTranscludeFn
  );
}
```

如果你需要在指令里面做一些复杂的事情，需要运行我们自建的、针对子节点的编译和链接操作，那另一个种“传递” transclusion 函数的方法就比较有用了。其中一个用例就是由`ng-if`指令控制下的“懒编译”指令。

当我们有这样的一个指令的话，并用在一个 transclusion 区域之中的时候可能会出问题，因为如果我们对它们进行分别链接，transclusion 函数无法查找到从父到子的联系。

你也可以让这种指令支持 transclusion，通过对自定义编译过程的节点的公共链接函数传递一个额外的参数。这个参数是一个`options`对象，而这个对象里面支持定义一个 key 为`parentBoundTranscludeFn`的属性。定义了这个属性的话，我们就可以把我们接收到的 transclusion 函数传递其他链接过程中去：

_test/compile\_spec.js_

```js
it('supports passing transclusion function to public link function', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function($compile) {
      return {
        transclude: true,
        link: function(scope, element, attrs, ctrl, transclude) {
          var customTemplate = $('<div in-custom-template></div>');
          element.append(customTemplate);
          $compile(customTemplate)(scope, {
            parentBoundTranscludeFn: transclude
          });
        }
      };
    },
    inCustomTemplate: function() {
      return {
        link: function(scope, element, attrs, ctrl, transclude) {
          element.append(transclude());
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div in-transclude></div></div>');

    $compile(el)($rootScope);

    expect(el.fnd('> [in-custom-template] > [in-transclude]').length).toBe(1);
  });
});
```

公共链接函数需要接收这个可选的`optioins`参数，如果能访问到`parentBoundTranscludeFn`属性的话，就把它的值拿出来。然后就可以把这个值传递给组合链接函数，这个函数已经允许把`parentBoundTranscludeFn`作为第三个参数：

_src/compile.js_

```js
return function publicLinkFn(scope, options) {
  options = options || {};
  var parentBoundTranscludeFn = options.parentBoundTranscludeFn;
  // $compileNodes.data('$scope', scope);
  compositeLinkFn(scope, $compileNodes, parentBoundTranscludeFn);
  // return $compileNodes;
};
```

> `options`实际上是公共链接函数的第三个参数，而不是第二个。而真正的第二个参数我们会在后面的章节中加入。

这样的话确实会带来一个有关 scope 生命周期的问题：我们传递过去的是一个绑定了作用域的 transclusion function，因为在链接函数里面有的就是这个。那 transclusion 作用域的`$parent`将会被误绑为当前的作用域，即使我们在这里没有链接过 transclusion。当我们销毁自定义链接的内容，并希望在经过 transclude 的内容里面的观察者都停止工作时，出现这种情况是很正常的（？）。

_test/compile\_spec.js_

```js
it('destroys scope passed through public link fn at the right time', function() {
  var watchSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myTranscluder: function($compile) {
      return {
        transclude: true,
        link: function(scope, element, attrs, ctrl, transclude) {
          var customTemplate = $('<div in-custom-template></div>');
          element.append(customTemplate);
          $compile(customTemplate)(scope, {
            parentBoundTranscludeFn: transclude
          });
        }
      };
    },
    inCustomTemplate: function() {
      return {
        scope: true,
        link: function(scope, element, attrs, ctrl, transclude) {
          element.append(transclude());
          scope.$on('destroyNow', function() {
            scope.$destroy();
          });
        }
      };
    },
    inTransclude: function() {
      return {
        link: function(scope) {
          scope.$watch(watchSpy);
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div in-transclude></div></div>');

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

我们需要搞清楚怎么在这个测试中对绑定了作用域的 transclusion 函数进行解绑。其实诀窍就是对这个函数添加一个属性，这个属性就指向这个函数所包裹在内的那个函数：

_src/compile.js_

```js
function scopeBoundTranscludeFn(transcludedScope) {
  return boundTranscludeFn(transcludedScope, scope);
}
scopeBoundTranscludeFn.$$boundTransclude = boundTranscludeFn;
```

然后在公共链接函数里，我们就可以在有这个属性时利用这个属性进行解绑。这就让一切都能回到正轨了：

```js
return function publicLinkFn(scope, options) {
  // options = options || {};
  // var parentBoundTranscludeFn = options.parentBoundTranscludeFn;
  if (parentBoundTranscludeFn && parentBoundTranscludeFn.$$boundTransclude) {
    parentBoundTranscludeFn = parentBoundTranscludeFn.$$boundTransclude;
  }
  // $compileNodes.data('$scope', scope);
  // compositeLinkFn(scope, $compileNodes, parentBoundTranscludeFn);
  // return $compileNodes;
};
```



