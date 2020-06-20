### $eval——在作用域语境下对表达式求值（$eval - Evaluating Code In The Context of A Scope）

在 Angular 中有几种途径可以在指定作用域的上下文内执行表达式。最简单的方法就是使用 `$eval`。这个方法需要接收一个函数作为参数，它会立即调用这个函数，调用时需要传入一个作用域。这个函数的返回值就是 `$eval` 方法的返回值。`$eval` 方法还可以接收一个额外的参数，这个参数会原封不动直接传递到 `$eval` 传入的函数中。

下面这几个测试用例展示了我们应该如何使用 `$eval`。我们把这几个测试用例都放到一个新的 `describe` 代码块中：

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

`$eval` 实现起来也很简单直接：

_src/scope.js_

```js
Scope.prototype.$eval = function(expr, locals) {
  return expr(this, locals);
};
```

那为什么我们调用一个函数也要这样绕圈子呢？可能有人会说，`$eval` 只是为了更明显地表现出这段代码是在特定作用域的上下文中执行的。`$apply` 也是基于 `$eval` 实现，我们后面会讲到。

但是 `$eval` 真正有趣的地方可能要到后面我们开始介绍表达式时才能真正展现出来，现在我们还在用原生函数形式的 `$eval` ，暂时还看不出来。到那时候，跟 `$watch` 一样，你可以直接传递一个字符串形式的表达式给 `$eval`。`$eval` 会对这个表达式进行编译，然后再在给定的作用域语境中执行它。我们会在本书第二部分实现这些功能。