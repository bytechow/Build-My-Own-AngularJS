### 解析 true, false 和 null
#### Parsing true, false, and null

我们需要支持的第三种字面量是布尔值字面量 `true` 和 `false`，还有 `null`。它们都是所谓的 _标识符_ token，这意味着它们在输入时是一个纯字母数字（alphanumeric）字符。随着不断开发，我们会遇到很多标识符。大多数情况它们用于通过名称查找作用域上的属性，但它们也可以是保留词，如果 `true`、`false` 或 `null`。在解析这三个 token 的时候，就让它们变成对应的 JavaScript 值就好了：

_test/parse_spec.js_

```js
it('will parse null', function() {
  var fn = parse('null');
  expect(fn()).toBe(null);
});

it('will parse true', function() {
  var fn = parse('true');
  expect(fn()).toBe(true);
});

it('will parse false', function() {
  var fn = parse('false');
  expect(fn()).toBe(false);
});
```

我们可以在 Lexer 中通过检查字符序列是否以大写或小写字母、下划线或美元符号开头来辨认字符是否是标识符：

_src/parse.js_

```js
Lexer.prototype.isIdent = function(ch) {
  return (ch >= 'a' && ch <= 'z') || (ch >= 'A' && ch <= 'Z') ||
    ch === '_' || ch === '$';
};
```

当我们遇到标志符，我们会使用一个名为 `readIdent` 的新函数对它进行解析：

_src/parse.js_

```js
Lexer.prototype.lex = function(text) {
  this.text = text;
  this.index = 0;
  this.ch = undefined;
  this.tokens = [];
  while (this.index < this.text.length) {
    this.ch = this.text.charAt(this.index);
    if (this.isNumber(this.ch) ||
        (this.ch === '.' && this.isNumber(this.peek()))) {
      this.readNumber();
    } else if (this.ch === '\'' || this.ch === '"') {
      this.readString(this.ch);
    } else if (this.isIdent(this.ch)) {
      this.readIdent();
    } else {
      throw 'Unexpected next character: '+this.ch;
    }
  }
  return this.tokens;
};
```