### 链式调用过滤器的表达式

#### Filter Chain Expressions

过滤器还有一个重要的特性，就是它支持链式调用。这意味着，我们可以使用多个过滤器，中间使用管道符号分割：

_test/parse\_spec.js_

```js
it('can parse filter chain expressions', function() {
  register('upcase', function() {
    return function(s) {
      return s.toUpperCase();
    };
  });
  register('exclamate', function() {
    return function(s) {
      return s + '!';
    };
  });
  var fn = parse('"hello" | upcase | exclamate');
  expect(fn()).toEqual('HELLO!');
});
```

现在，我们会在使用第一个过滤器后就终止表达式。

实际上，我们只需要把 `AST.prototype.filter` 中的 `if` 语句换成 `while` 语句就可以了。只要我们发现表达式后面有管道符号，我们就会应用下一个过滤器。其中一个过滤器的 `CallExpression` 的结果值会成为下一个的参数：

_src/parse.js_

```js
AST.prototype.filter = function() {
  // var left = this.assignment();
  while (this.expect('|')) {
  //   left = {
  //     type: AST.CallExpression,
  //     callee: this.identifier(),
  //     arguments: [left],
  //     filter: true
  //   };
  // }
  // return left;
};
```



