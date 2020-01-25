### 简单的属性查找
#### Simple Attribute Lookup

访问作用域属性最简单的方式就是按属性名称检索：表达式 `‘aKey’` 会从作用域对象中找到 `aKey` 属性，然后返回它：

_test/parse_spec.js_

```js
it('looks up an attribute from the scope', function() {
  var fn = parse('aKey');
  expect(fn({aKey: 42})).toBe(42);
  expect(fn({})).toBeUndefined();
});
```

注意，`parse` 函数的返回值是一个函数，这个函数会接受一个 JavaScript 对象作为参数。这个对象一般都是 `Scope` 的实例，这个实例能被表达式访问或操作。但它不一定非得是一个作用域，在单元测试中，我们可以直接用普通的对象字面量。由于字面量表达式跟作用域没有半点关系，之前我们都没有使用过这个参数，但在本章情况会发生变化。事实上，我们要做的第一个改变就是在嘴中编译生成的表达式函数中加入这个参数。我们把它称为 `s`：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  var ast = this.astBuilder.ast(text);
  this.state = { body: [] };
  this.recurse(ast);
  /* jshint -W054 */
  return new Function('s', this.state.body.join(''));
  /* jshint +W054 */
};
```