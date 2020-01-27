### 解析 this
#### Parsing this

还有一种特殊的属性检索方式需要我们进行特殊的处理，就是对 `this` 的引用。`this` 在 Angular 表达式中的角色与它在 JavaScript 中的角色类似：那就是运算表达式的上下文。表达式函数的上下文总是指向它运算时传入的作用域，因此 `this` 就是指向这个作用域：

_test/parse_spec.js_

```js
it('will parse this', function() {
  var fn = parse('this');
  var scope = {};
  expect(fn(scope)).toBe(scope);
  expect(fn()).toBeUndefined();
});
```