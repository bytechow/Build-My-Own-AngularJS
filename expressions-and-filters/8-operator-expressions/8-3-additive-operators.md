### 加法类运算符
#### Additive Operators

紧接着乘法类运算符的是加法类运算符：加号和减号。我们已经在一元运算符的章节中使用过这两个符号了，现在需要把它们看作一个二元运算符了：

_test/parse_spec.js_

```js
it('parses an addition', function() {
  expect(parse('20 + 22')()).toBe(42);
});

it('parses a subtraction', function() {
  expect(parse('42 - 22')()).toBe(20);
});
```

如上所述，加法类运算符的优先级紧跟着乘法类运算符：

_test/parse_spec.js_

```js
it('parses multiplicatives on a higher precedence than additives', function() {
  expect(parse('2 + 3 * 5')()).toBe(17);
  expect(parse('2 + 3 * 2 + 3')()).toBe(11);
});
```

我们已经在 `OPERATORS` 对象中涵盖了这两个运算符了。而在构建 AST 时，我们需要新建一个 `additive` 函数，它看上去跟 `multiplicative` 差不多，只是它所“期待”的运算符字符和要调用的下一个函数不同而已：

_src/parse.js_

```js
AST.prototype.additive = function() {
  var left = this.multiplicative();
  var token;
  while ((token = this.expect('+')) || (token = this.expect('-'))) {
    left = {
      type: AST.BinaryExpression,
      left: left,
      operator: token.text,
      right: this.multiplicative()
    };
  }
  return left;
};
```

加法类运算符的执行顺序在赋值运算符和乘法类运算符之间，这意味着 `assignment` 应该调用 `additive`，而 `additive` 会调用 `multiplicative`：

_src/parse.js_

```js
AST.prototype.assignment = function() {
  var left = this.additive();
  // if (this.expect('=')) {
    var right = this.additive();
  //   return { type: AST.AssignmentExpression, left: left, right: right };
  // }
  // return left;
};
```

由于我们在处理乘法类运算符时已经实现了 AST 编译器对二元表达式（binary expressions）的支持，因此以上测试用例直接就能通过了。从编译器的视角来看，加法类运算符和乘法类运算符并没有什么不一样，除了：

正如我们在实现一元运算符时看到的那样，如果 `+` 和 `-` 对应的运算数缺失，这个运算符会被视作 0。这个规则也适用于加法和乘法运算。如果两旁的运算数有一个或都为空，都会使用 0 替代。

_test/parse_spec.js_

```js
it('substitutes undefined with zero in addition', function() {
  expect(parse('a + 22')()).toBe(22);
  expect(parse('42 + a')()).toBe(42);
});

it('substitutes undefined with zero in subtraction', function() {
  expect(parse('a - 22')()).toBe(-22);
  expect(parse('42 - a')()).toBe(42);
});
```

在编译时，对于加法或减法传入的参数，我们都应该用 `isDefined` 进行包裹，但也仅仅是加法和减法而已：

_src/parse.js_

```js
case AST.BinaryExpression:
  if (ast.operator === '+' || ast.operator === '-') {
    return '(' + this.ifDefined(this.recurse(ast.left), 0) + ')' +
      ast.operator +
      '(' + this.ifDefined(this.recurse(ast.right), 0) + ')';
  } else {
    // return '(' + this.recurse(ast.left) + ')' + ast.operator +
    //   '(' + this.recurse(ast.right) + ')';
  }
  break;
```