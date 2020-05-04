### 字面量与常量表达式
#### Literal And Constant Expressions

现在我们已经知道解析器是如何返回一个用于计算原始表达式的函数的了。已返回的函数并不仅仅是一个普通函数，它需要添加一些额外的属性：

- `literal` —— 一个布尔值，用于说明表达式是否为字面量，例如整数或者数组字面量。
- `constant` —— 一个布尔值，用于说明表达式是否为一个常量（例如一个字面量原始类型），或者一个字面亮常量的集合。如果表达式是常量，它将永远保持不变。

比如，`42` 既是一个字面量，也是一个常量，`[42, 'abc']` 也是一样的。而像 `[42, 'abc', aVariable]` 这种虽然是一个字面量，但由于 `aVariable` 是一个变量，所以这个表达式就不是一个常量了。

`$parse` 服务的使用者有时会使用这两个标志来决定如何使用表达式。本章将会利用 `constant` 标志来对表达式监听做一些优化。

> 实际上，解析器返回的函数还有第三个属性 `assign`，本章后面会讲到。

我们会先介绍更容易实现的 `literal` 标识。所有简单的原始类型字面量，包括数字、字符串和布尔值，都会被标记为字面量：

_test/parse_spec.js_

```js
it('marks integers literal', function() {
  var fn = parse('42');
  expect(fn.literal).toBe(true);
});

it('marks strings literal', function() {
  var fn = parse('"abc"');
  expect(fn.literal).toBe(true);
});

it('marks booleans literal', function() {
  var fn = parse('true');
  expect(fn.literal).toBe(true);
});
```

数组和对象字面量也一样：

_test/parse_spec.js_

```js
it('marks arrays literal', function() {
  var fn = parse('[1, 2, aVariable]');
  expect(fn.literal).toBe(true);
});

it('marks objects literal', function() {
  var fn = parse('{a: 1, b: aVariable}');
  expect(fn.literal).toBe(true);
});
```

其它值都会被标记为“非字面量”（non-literal）：

_test/parse_spec.js_

```js
it('marks unary expressions non-literal', function() {
  var fn = parse('!false');
  expect(fn.literal).toBe(false);
});

it('marks binary expressions non-literal', function() {
  var fn = parse('1 + 2');
  expect(fn.literal).toBe(false);
});
```

我们需要使用一个辅助函数 `isLiteral` 来检查 AST 是否是一个字面量。然后我们会将检查结果作为编译后的表达式函数的一个属性：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = {
  //   body: [],
  //   nextId: 0,
  //   vars: [],
  //   filters: {}
  // };
  // this.recurse(ast);
  // var fnString = this.filterPrefix() +
  //   'var fn=function(s,l){' + (this.state.vars.length ?
  //     'var ' + this.state.vars.join(',') + ';' :
  //     ''
  //   ) + this.state.body.join('') + '}; return fn;';
  /* jshint -W054 */
  var fn = new Function(
    // 'ensureSafeMemberName',
    // 'ensureSafeObject',
    // 'ensureSafeFunction',
    // 'ifDefined',
    // 'filter',
    // fnString)(
    //   ensureSafeMemberName,
    //   ensureSafeObject,
    //   ensureSafeFunction,
    //   ifDefined,
    //   filter);
  /* jshint +W054 */
  fn.literal = isLiteral(ast);
  return fn;
};
```