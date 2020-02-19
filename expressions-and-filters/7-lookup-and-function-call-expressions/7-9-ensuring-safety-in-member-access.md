### 在访问对象成员时启用安全策略
#### Ensuring Safety In Member Access

由于表达式一般会被嵌入到 HTML 中，而且嵌入的内容很可能会加入用户生成内容（user-generated content），也就是说，用户有可能通过生成特定表达式的方法来达到在程序中执行任意代码的效果，因此防止注入攻击是我们的重中之重。对此产生的防护措施主要基于一个原理，我们可以把所有表达式的上下文严格限制在 Scope 对象以内：除了字面量，我们只能访问添加在 scope 对象上的数据（或者 locals 上的数据）。存在安全隐患的对象，比如 `window`，我们会直接禁止访问。

在我们当前已经实现的代码中有几种方法可以解决这个问题：尤其是，在本章中，我们已经看到 JavaScript 的 `Function` 构造器是如何接收一个字符串，然后把这个字符串作为一个新函数的源代码。我们会使用这个新函数来生成表达式函数。事实证明，如果我们不采取限制，表达式确实可以利用同一个 Function 构造函数来运行任意的代码。

由于每一个 JavaScript 函数的 `constructor` 属性都指向 Function 构造函数，这让攻击者有机可乘。假如作用域上有一个函数（这很常见），我们可以在表达式中访问到它的构造函数，只要给它传递一段 JavaScript 代码，然后执行生成的函数。到这时，一切就完了。比如，我们可以轻易地拿到全局的 `window` 对象：

```js
aFunction.constructor('return window;')()
```

除了 Function 构造函数，还有一些常见的对象也可能会引发安全问题，允许访问它们可能会产生难以预测的影响：

- `__proto__` 是一个[非标准的、已被弃用的对象原型访问器](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/proto)。它不仅允许读取原型，还可以对原型进行设置，这种特性使得它也存在潜在危险。

- `__defineGetter__, __lookupGetter__, __defineSetter__,`and`__lookupSetter__` 是[根据 getter 和 setter 函数来对对象属性进行定义的非标准函数](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/__defineGetter__)。由于它们并不是标准的 API，并不被所有的浏览器支持，且它们也有重新定义全局属性的可能性，Angular 会直接在表达式中禁止访问这些函数。

下面我们来加入单元测试，确保在表达式中不能访问以上六个成员：

_test/parse_spec.js_

```js
it('does not allow calling the function constructor', function() {
  expect(function() {
    var fn = parse('aFunction.constructor("return window;")()');
    fn({ aFunction: function() {} });
  }).toThrow();
});

it('does not allow accessing __proto__', function() {
  expect(function() {
    var fn = parse('obj.__proto__');
    fn({ obj: {} });
  }).toThrow();
});

it('does not allow calling __defineGetter__', function() {
  expect(function() {
    var fn = parse('obj.__defineGetter__("evil", fn)');
    fn({ obj: {}, fn: function() {} });
  }).toThrow();
});

it('does not allow calling __defineSetter__', function() {
  expect(function() {
    var fn = parse('obj.__defineSetter__("evil", fn)');
    fn({ obj: {}, fn: function() {} });
  }).toThrow();
});

it('does not allow calling __lookupGetter__', function() {
  expect(function() {
    var fn = parse('obj.__lookupGetter__("evil")');
    fn({ obj: {} });
  }).toThrow();
});

it('does not allow calling __lookupSetter__', function() {
  expect(function() {
    var fn = parse('obj.__lookupSetter__("evil")');
    fn({ obj: {} });
  }).toThrow();
});
```

针对这类攻击，我们将要采取的安全措施是禁止访问对象上任何具有上述成员名称的属性。