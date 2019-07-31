###  克隆 DOM 的 Attach 函数（The Clone Attach Function）

我们之前已经看到我们在指令链接函数（或控制器）中获取的 transclusion 函数是如何运作的：调用它，就会返回一个指向进行了 transclude 的 DOM 的引用，利用这个引用之后就可以在某处插入这个 DOM。而在函数内部，他会把经过 transclude 的内容链接到一个 transclusion 作用域上去。这个作用域的父作用域是什么取决于 transclusion 是在哪里定义和是在哪里调用的。你也可以选择把一个作用域作为参数传递给 transclusion 函数，这样它就会使用这个作用域进行链接，而不需要再构建一个 transclusion 作用域。

对于 transclusion 函数，我们还需要补充一个核心的知识点。我们称为_clone attach function_。

克隆的 Attach 函数，就是我们在链接 DOM 片段时随时都可以提供的一个函数。当我们提供了一个 clone attach function  时，Angular 并不会链接已经经过编译的原始 DOM，而是会先先创建一个 DOM 片段的克隆，然后对克隆进行链接。它也会在编译期间调用这个函数，克隆后的 DOM 和作用域都会提供给这个函数进行链接。在那个节点，我们需要会把克隆的 DOM 添加到某处，所以才会命名为“clone attach function”。

所以，clone attach function 有两个关联但又互相独立的用途：

1. 作为副作用，使用一个 clone attach function  ，就会创建并链接一段经过编译的 DOM 克隆。
2. 它充当链接阶段的一个回调，它会在所有 DOM 都克隆好了之后才会调用。

之后，我们就会看到这两个用途的用处。

 clone attach function 的核心就是它并不需要跟 transclusion 有关联。它是公共链接 API 的一部分，而且即使没有 transclusion 也可以使用。我们会把这两个知识点都放到这个章节介绍，因为它们俩经常都是一起出现的。

当我们在公共链接函数中加入 clone attach function 时，它会在链接阶段被调用：

_test/compile_spec.js_

```js
describe('clone attach function', function() {
  it('can be passed to public link fn', function() {
    var injector = makeInjectorWithDirectives({});
    injector.invoke(function($compile, $rootScope) {
      var el = $('<div>Hello</div>');
      var myScope = $rootScope.$new();
      var gotEl, gotScope;

      $compile(el)(myScope, function cloneAttachFn(el, scope) {
        gotEl = el;
        gotScope = scope;
      });
      
      expect(gotEl[0].isEqualNode(el[0])).toBe(true);
      expect(gotScope).toBe(myScope);
    });
  });
});
```

我们可以通过调用从公共链接函数提供的 clone attach function  来通过这个单元测试。我们会在链接 DOM 之前进行调用：

_src/compile.js_

```js
return function publicLinkFn(scope, cloneAttachFn, options) {
  // options = options || {};
  // var parentBoundTranscludeFn = options.parentBoundTranscludeFn;
  // if (parentBoundTranscludeFn && parentBoundTranscludeFn.$$boundTransclude) {
  //   parentBoundTranscludeFn = parentBoundTranscludeFn.$$boundTransclude;
  // }
  // $compileNodes.data('$scope', scope);
  if (cloneAttachFn) {
    cloneAttachFn($compileNodes, scope);
  }
  // compositeLinkFn(scope, $compileNodes, parentBoundTranscludeFn);
  // return $compileNodes;
};
```

因为我们会把 clone attach function 作为第二个参数，那之前介绍的`options`参数就会顺移到第三个位置。为此，我们需要修改之前的两个单元测试`“supports passing transclusion function to public link function”`和`“destroys scope passed through public link
fn at the right time”`。也就是在这两个测试用例中，我们会把`undefined`作为 clone attach function：

_src/compile.js_

```js
$compile(customTemplate)(scope, undefned, {
  parentBoundTranscludeFn: transclude
});
```

正如我们讨论的这样，它不再是一个普通的回调函数，因为它会对原有的 DOM 进行克隆：

_test/compile_spec.js_

```js
it('causes compiled elements to be cloned', function() {
  var injector = makeInjectorWithDirectives({});
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div>Hello</div>');
    var myScope = $rootScope.$new();
    var gotClonedEl;

    $compile(el)(myScope, function(clonedEl) {
      gotClonedEl = clonedEl;
    });
    
    expect(gotClonedEl[0].isEqualNode(el[0])).toBe(true);
    expect(gotClonedEl[0]).not.toBe(el[0]);
  });
})
```

这段克隆的 DOM 不仅仅只用于 clone attach function，它也是经过链接的 DOM 的版本。换句话说，原本的 DOM 将不会进行链接。因此在这种情况下，指令链接函数接收到的 DOM 元素将不同于它编译函数接收到的：

```js
it('causes cloned DOM to be linked', function() {
  var gotCompileEl, gotLinkEl;
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        compile: function(compileEl) {
          gotCompileEl = compileEl;
          return function link(scope, linkEl) {
            gotLinkEl = linkEl;
          };
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    var myScope = $rootScope.$new();

    $compile(el)(myScope, function() {});
    
    expect(gotCompileEl[0]).not.toBe(gotLinkEl[0]);
  });
});
```

因此，如果如果提供了一个 clone attach function 的话，我们应该对被编译的节点进行克隆，然后将这个克隆的节点传递给 clone attach function 和组合链接函数，而不是原始的要进行编译的节点：

_src/compile.js_

```js
return function publicLinkFn(scope, cloneAttachFn, options) {
  // options = options || {};
  // var parentBoundTranscludeFn = options.parentBoundTranscludeFn;
  // if (parentBoundTranscludeFn && parentBoundTranscludeFn.$$boundTransclude) {
  //   parentBoundTranscludeFn = parentBoundTranscludeFn.$$boundTransclude;
  // }
  var $linkNodes;
  if (cloneAttachFn) {
    $linkNodes = $compileNodes.clone();
    cloneAttachFn($linkNodes, scope);
  } else {
    $linkNodes = $compileNodes;
  }
  $linkNodes.data('$scope', scope);
  compositeLinkFn(scope, $linkNodes, parentBoundTranscludeFn);
  return $linkNodes;
};
```

这样我们的测试用例就可以通过了。但应该注意的是，我们同样也会对克隆节点的`$scope`这个 jQuery 添加的数据进行改变，因为这些节点才是最终会绑定到作用域上的。

如果传递了一个 clone attach function 就需要在链接之前对节点进行克隆，但这还是没有解释到为什么我们需要为此创建一个函数。难道用一个布尔类型的标识代替不行吗？对于当前的代码开发进度来说，确实是可以这样代替，但当我们重新回到 transclusion 的话题中时，我们使用函数就会变得比较合理了。

首先，你可以传递一个 clone attach function 给 transclusion 函数。当你这样做的时候，它会提供一种获取 transclude 内容的替代方法。它们不仅作为 transclusion 函数的返回值，同时这些内容会成为 clone attach function 的第一个参数。这意味着我们可以做这样的事情：

_test/compile_spec.js_

```js
it('allows connecting transcluded content', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true,
        template: '<div in-template></div>',
        link: function(scope, element, attrs, ctrl, transcludeFn) {
          var myScope = scope.$new();
          transcludeFn(myScope, function(transclNode) {
            element.find('[in-template]').append(transclNode);
          });
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div in-transclude></div></div>');

    $compile(el)($rootScope);
    
    expect(el.find('> [in-template] > [in-transclude]').length).toBe(1);
  });
});
```

指令中调用的是绑定了 scope 的 transclusion 函数，所以我们需要让它能够接收 clone attach function。它可以继续被传递到 inner bound transclusion function，同样也是作为第二个参数：

_src/compile.js_

```js
function scopeBoundTranscludeFn(transcludedScope, cloneAttachFn) {
  return boundTranscludeFn(transcludedScope, cloneAttachFn, scope);
}
// scopeBoundTranscludeFn.$$boundTransclude = boundTranscludeFn;
```

inner bound transclusion function 接收到这个参数，然后将它传递给真正的 transclusion 函数，也就是已经对 clone attach function 进行支持的公共链接函数：

```js
var boundTranscludeFn;
// if (linkFn.nodeLinkFn.transcludeOnThisElement) {
  boundTranscludeFn = function(transcludedScope, cloneAttachFn, containingScope) {
    // if (!transcludedScope) {
    //   transcludedScope = scope.$new(false, containingScope);
    // }
    return linkFn.nodeLinkFn.transclude(transcludedScope, cloneAttachFn);
  };
// } else if (parentBoundTranscludeFn) {
//   boundTranscludeFn = parentBoundTranscludeFn;
// }
```

但这依然无法解释为什么我们需要一个函数。传递一个布尔值标识也能够满足目前的测试用例。

有一个解释是跟执行时机有关：当你获取了 transclusion function 的返回值，DOM 其实已经被链接了。但 clone attach function 是在链接之前调用的。这让我们在链接之前还有机会对克隆后的 DOM 进行操作，就在 clone attach function 中进行改变。

但 clone attach function 的存在还有一个更大的原因，是跟 scope 有关的，我们将会在稍后进行介绍。