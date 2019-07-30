###  clone attach function  （The Clone Attach Function）

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

因为我们会把 clone attach function  作为第二个参数，那之前介绍的`options`参数就会顺移到第三个位置。为此，我们需要修改之前的两个单元测试`“supports passing transclusion function to public link function”`和`“destroys scope passed through public link
fn at the right time”`。也就是在这两个测试用例中，我们会把`undefined`作为 clone attach function：

_src/compile.js_

```js
$compile(customTemplate)(scope, undefned, {
  parentBoundTranscludeFn: transclude
});
```