### 逻辑并和逻辑非运算符
#### Logical Operators AND and OR

接下来，我们要实现的两个逻辑操作符是 `&&` 和 `||`。它们的功能正如我们所期待的那样：

_test/parse_spec.js_

```js
it('parses logical AND', function() {
  expect(parse('true && true')()).toBe(true);
  expect(parse('true && false')()).toBe(false);
});

it('parses logical OR', function() {
  expect(parse('true || true')()).toBe(true);
  expect(parse('true || false')()).toBe(true);
  expect(parse('false || false')()).toBe(false);
});
```

跟其他二元运算一样，我们可以连续使用多个逻辑运算符：

_test/parse_spec.js_

```js
it('parses logical AND', function() {
  expect(parse('true && true')()).toBe(true);
  expect(parse('true && false')()).toBe(false);
});

it('parses logical OR', function() {
  expect(parse('true || true')()).toBe(true);
  expect(parse('true || false')()).toBe(true);
  expect(parse('false || false')()).toBe(false);
});
```

逻辑运算符的有趣之处是它们拥有的“短路”特性。如果逻辑并表达式的值为假值（falsy），那就不会再执行右侧的表达式了，跟它们在 JavaScript 的表现一模一样：

_test/parse_spec.js_

```js
it('short-circuits AND', function() {
  var invoked;
  var scope = { fn: function() { invoked = true; } };

  parse('false && fn()')(scope);
  
  expect(invoked).toBeUndefined();
});
```

而对于逻辑或运算符，如果它左侧的表达式结果为真值（truthy），那右侧表达式就不会被执行：

_test/parse_spec.js_

```js
it('short-circuits OR', function() {
  var invoked;
  var scope = { fn: function() { invoked = true; } };

  parse('true || fn()')(scope);
  
  expect(invoked).toBeUndefined();
});
```

逻辑并的优先级高于逻辑或：

_test/parse_spec.js_

```js
it('parses AND with a higher precedence than OR', function() { expect(parse('false && true || true')()).toBe(true);
});
```

这里我们测试的表达式的运算结果将会是 `(false && true) || true` 而不是 `false && (true || true)`。

相等性运算符的优先级高于逻辑运算符：

_test/parse_spec.js_

```js
it('parses OR with a lower precedence than equality', function() { expect(parse('1 === 2 || 2 === 2')()).toBeTruthy();
});
```