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

复合链接函数会接收两个参数：