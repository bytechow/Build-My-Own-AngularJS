### 三元运算符
#### The Ternary Operator

本章要实现的最后一个运算符（以及倒数第二个操作符）是 C 语言风格的三元运算符，这个运算符让我们可以根据测试表达式的结果来选择返回两个可选值中的哪一个：

_test/parse_spec.js_

```js
it('parses the ternary expression', function() {
  expect(parse('a === 42 ? true : false')({ a: 42 })).toBe(true);
  expect(parse('a === 42 ? true : false')({ a: 43 })).toBe(false);
});
```

三元运算符的优先级刚好比逻辑或（OR）运算符低一级，因此，我们会先运算逻辑或表达式：

_test/parse_spec.js_

```js
it('parses OR with a higher precedence than ternary', function() {
  expect(parse('0 || 1 ? 0 || 2 : 0 || 3')()).toBe(2);
});
```

我们也可以使用嵌套的三元运算符，虽然这样做可能会让代码结构变得不清晰：

_test/parse_spec.js_

```js
it('parses nested ternaries', function() {
  expect(
    parse('a === 42 ? b === 42 ? "a and b" : "a" : c === 42 ? "c" : "none"')({
      a: 44,
      b: 43,
      c: 42
    })).toEqual('c');
});
```

跟本章看到的大多数运算符不一样，三元运算符并不会以一个运算符函数的形式出现在 `OPERATORS` 对象中。这是因为三元运算表达式中会有两个不同的运算符：问号`？`和冒号`:`，这样便于我们在 AST 构建阶段进行检测。