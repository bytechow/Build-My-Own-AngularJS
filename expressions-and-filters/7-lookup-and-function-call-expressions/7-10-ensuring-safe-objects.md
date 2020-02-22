### 在访问对象时启用安全策略
#### Ensuring Safe Objects

Angular 表达式提供的第二个安全措施更多的是为了保护应用开发者免受自己的攻击，具体来说就是不让他们在作用域上添加危险的对象，然后在表达式访问它们。

其中一个危险的对象就是 window。调用 `window` 上的某些函数可能会带来巨大的破坏，所以 Angular 将完全阻止你在表达式中使用它。当然，你不能直接调用 `window` 上的成员，因为表达式只能在作用域上下文范围内进行访问，但需要保证 `window` 不会变成作用域上的一个同名属性。如果你试图这样做，就会抛出一个异常：

_test/parse_spec.js_

```js
it('does not allow accessing window as computed property', function() { var fn = parse('anObject["wnd"]');
  expect(function() { fn({anObject: {wnd: window}}); }).toThrow();
});

it('does not allow accessing window as non-computed property', function() { var fn = parse('anObject.wnd');
  expect(function() { fn({anObject: {wnd: window}}); }).toThrow();
});
```

相应的安全措施就是当我们处理对象时，需要确保它们里面不包含危险的对象。为此，我们先加入一个帮助函数：

_src/parse.js_

```js
function ensureSafeObject(obj) {
  if (obj) {
    if (obj.window === obj) {
    throw 'Referencing window in Angular expressions is disallowed!';
    }
  }
  return obj;
}
```

我们检查对象是否不包含 window 的方式是看这个对象是否有一个 `window` 属性，并且这个属性指向自身——在 JavaScript 中，这种特性只有 `window` 对象有，其他对象没有。

要让测试通过，我们需要先在运行时加入这个新的帮助函数：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  // this.state = { body: [], nextId: 0, vars: [] };
  // this.recurse(ast);
  // var fnString = 'var fn=function(s,l){' + (this.state.vars.length ?
  //   'var ' + this.state.vars.join(',') + ';' :
  //   ''
  // ) + this.state.body.join('') + '}; return fn;';
  /* jshint -W054 */
  return new Function(
    'ensureSafeMemberName',
    'ensureSafeObject',
    fnString)(
      ensureSafeMemberName,
      ensureSafeObject);
  /* jshint +W054 */
};
```