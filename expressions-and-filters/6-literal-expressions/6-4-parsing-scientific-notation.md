### 解析科学记数法
#### Parsing Scientific Notation

第三种，也是最后一种在 Angular 表达式表示数字的方法是科学记数，科学记数由两个数字构成：系数（coefficient）和指数（exponent），它们之间通过字符 `e` 进行分割。举个例子，数字 `42,000`，或者 `42 * 10³`，都能用 `42e3` 来表示。下面是对应的单元测试：

_test/parse_spec.js_

```js
it('can parse a number in scientific notation', function() {
  var fn = parse('42e3');
  expect(fn()).toBe(42000);
});
```

另外，科学记数法中的系数也不一定就是一个整数：

_test/parse_spec.js_

```js
it('can parse scientific notation with a float coefficient', function() {
  var fn = parse('.42e2');
  expect(fn()).toBe(42);
});
```

而科学记数法中的指数也可能是负数，让系数乘以负的 10 次方：

_test/parse_spec.js_

```js
it('can parse scientific notation with negative exponents', function() {
  var fn = parse('4200e-2');
  expect(fn()).toBe(42);
});
```

指数也能被显式地标识为一个正数，只要在前面加一个 `+` 号就好了：

_test/parse_spec.js_

```js
it('can parse scientific notation with the + sign', function() {
  var fn = parse('.42e+2');
  expect(fn()).toBe(42);
});
```