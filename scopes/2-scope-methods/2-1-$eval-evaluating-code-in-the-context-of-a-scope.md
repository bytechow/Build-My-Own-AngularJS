### $eval——在作用域语境下对表达式求值（$eval - Evaluating Code In The Context of A Scope）

在 Angular 中，我们有几种途径可以使表达式在某个作用域语境下执行。最简单的方法就是使用 `$eval`。这个方法会接收一个函数作为参数，然后马上调用这个函数，同时把当前作用域作为这个函数的参数。`$eval` 方法的返回值就是它接收的函数的返回值。`$eval` 方法还可以接收一个额外的参数，这个参数会原封不动直接传递到那个函数中。

下面这几个测试用例就说明了我们应当如何使用 `$eval`。我们会把这几个测试用例都放到一个新的 `describe` 代码块中：

_test/scope_spec.js_

```js
describe('$eval', function() {
  
  var scope;

  beforeEach(function() {
    scope = new Scope();
  });
  
  it('executes $evaled function and returns result', function() {
    scope.aValue = 42;
  
    var result = scope.$eval(function(scope) {
      return scope.aValue;
    });
  
    expect(result).toBe(42);
  });
  
  it('passes the second $eval argument straight through', function() {
    scope.aValue = 42;
  
    var result = scope.$eval(function(scope, arg) {
      return scope.aValue + arg;
    }, 2);
  
    expect(result).toBe(44);
  });
  
});
```

开发 `$eval` 的过程也是很简单直接的：

_src/scope.js_

```js
Scope.prototype.$eval = function(expr, locals) {
  return expr(this, locals);
};
```

但我们为什么要这样绕着圈子来调用一个函数呢？有人可能会说，使用 `$eval` 能更清晰地展示出这段代码是在一个作用域的上下文中执行的。`$eval` 也是 `$apply` 的基本构成要素，这个我们在后面会讲到。

但是，可能要到后面我们开始介绍表达式时，才能展现 `$eval` 最有趣的用法，而非现在这种直接使用原生函数的形式。到那时候，跟 `$watch` 一样，你可以传递一个字符串给 `$eval`。`$eval` 会对这个表达式进行编译，然后再在给定的作用域语境中执行它。我们会在本书第二部分实现这一功能。