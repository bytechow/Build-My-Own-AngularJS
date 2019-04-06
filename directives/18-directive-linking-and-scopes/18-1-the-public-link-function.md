公开的 Link 函数（The Public Link Function）

一般来说，对一个棵 DOM 树应用 Angular 指令需要两个步骤：

1. 编译 DOM 树
2. 把编译后的 DOM 树与作用域进行链接

第一步我们已经介绍过了，所以现在我们会把焦点放到第二步上。

我们有一个名为`$compile`的服务用于编译了，所以可能会有人觉得会有一个名为`$link`的服务来进行链接。但实际上并不是这样的。Angular 并没有顶层的接口用于指令链接，这个过程也会被放到`compile.js`中。

编译和链接的实现代码虽然都放在一个文件中，但这两个过程实际上是独立开来的。当我们调用`$compile`时，并不会发生链接，但它会返回一个函数，这个函数可以用于在稍后初始化一个链接的过程，这个函数被称为公开的链接函数：

_test/compile\_spec.js_

```js
it('returns a public link function from compile', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: _.noop
    };
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive></div>');
    var linkFn = $compile(el);
    expect(linkFn).toBeDefned();
    expect(_.isFunction(linkFn)).toBe(true);
  });
});
```

所以，我们需要在`$compile`服务中对外公开的 compile 函数里返回这样的一个函数：

```js
function compile($compileNodes) {
  compileNodes($compileNodes);
  return function publicLinkFn() {};
}
```

那这个函数会干些什么呢？这个函数需要做很多工作，这一点我们稍后会看到，但它需要做的第一件事是对 DOM 添加一些调试信息。具体来说，这个函数会接收一个作用域对象作为参数，并使用 jQuery 或 jqLite 的 data 方法把作用域对象变成 DOM 节点的一个特定数据。

让我们先创建一个新的`describe`代码块，这个测试模块是专门针对指令链接的：

_test/compile\_spec.js_

```js
describe('linking', function() {
  it('takes a scope and attaches it to elements', function() {
    var injector = makeInjectorWithDirectives('myDirective', function() {
      return {
        compile: _.noop
      };
    });
    injector.invoke(function($compile, $rootScope) {
      var el = $('<div my-directive></div>');
      $compile(el)($rootScope);
      expect(el.data('$scope')).toBe($rootScope);
    });
  });
});
```

在这个测试用例中，我们会把`$rootScope`作为参数传递到公开的链接函数中去，并确认它是否会成为传入到`$compile`的元素的数据属性，这个数据属性名称为`$scope`。

要通过这个测试十分简单。我们可以直接对传入`compile`的元素添加这个数据即可：

_src/compile.js_

```js
function compile($compileNodes) {
  compileNodes($compileNodes);
  return function publicLinkFn(scope) {
    $compileNodes.data('$scope', scope);
  };
}
```

> Angular 中对 DOM 节点附加不同的数据属性和样式类的默认功能是可以关闭的，只需要调用`$compileProvider`的`debugInfoEnabled`即可。我们在生产环境中大多时候是不需要这种调试信息，主动禁用掉可以让应用提升一点性能，这是 Angular 允许我们这样设置的原因。由于这个优化过程与我们要讨论的东西并不密切相关，本书将会跳过这部分，不进行讲解。



