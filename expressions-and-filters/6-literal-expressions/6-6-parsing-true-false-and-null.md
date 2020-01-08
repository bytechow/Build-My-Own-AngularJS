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
  // this.text = text;
  // this.index = 0;
  // this.ch = undefined;
  // this.tokens = [];
  // while (this.index < this.text.length) {
  //   this.ch = this.text.charAt(this.index);
  //   if (this.isNumber(this.ch) ||
  //       (this.ch === '.' && this.isNumber(this.peek()))) {
  //     this.readNumber();
  //   } else if (this.ch === '\'' || this.ch === '"') {
  //     this.readString(this.ch);
    } else if (this.isIdent(this.ch)) {
      this.readIdent();
  //   } else {
  //     throw 'Unexpected next character: '+this.ch;
  //   }
  // }
  // return this.tokens;
};
```

一旦进入 `readIdent` 方法，我们将会用类似处理字符串的方式来读取标识符 token：

_src/parse.js_

```js
Lexer.prototype.readIdent = function() {
  var text = '';
  while (this.index < this.text.length) {
    var ch = this.text.charAt(this.index);
    if (this.isIdent(ch) || this.isNumber(ch)) {
      text += ch;
    } else {
      break; 
    }
    this.index++;
  }

  var token = {text: text};
  
  this.tokens.push(token);
};
```

要注意的是，标识符虽然可以包含数字，但不能以数字开头。

现在我们虽然有标识符 token，但它们还没转化成能被 AST builder 处理的数据类型。我们需要对此进行改变，可以通过让 AST 记录预定义字面量对应的“常量” token 来解决这个问题：

_src/parse.js_

```js
AST.prototype.constants = {
  'null': {type: AST.Literal, value: null},
  'true': {type: AST.Literal, value: true},
  'false': {type: AST.Literal, value: false}
};
```

要把这些常量插入到 AST 中，我们需要借助一个介于 `program` 和 `constant` 之间的中间函数 `primary`。也就是说，一个 program 体中包含一个 `primary` token：

_src/parse.js_

```js
AST.prototype.program = function() {
  return {type: AST.Program, body: this.primary()};
};

AST.prototype.primary = function() {
  return this.constant();
};

AST.prototype.constant = function() {
  return {type: AST.Literal, value: this.tokens[0].value};
};
```

一个 primary token 既可以是我们预定义的常量，也可以是在之前代码中定义的其他常量。

_src/parse.js_

```js
AST.prototype.primary = function() {
  if (this.constants.hasOwnProperty(this.tokens[0].text)) {
    return this.constants[this.tokens[0].text];
  } else {
    // return this.constant();
  }
};
```

现在关于 `true` 和 `false` 的单元测试都能通过了，因为它们都能被正常解析成对应的 JavaScript 值了。但对于 `null` 的单元测试还没通过，这是因为 `null` 默认的字符串表示是一个空字符串。在 compiler 的 `escape` 方法中，我们需要加入一个特殊的用例判断，以便 `null` 可以正常出现在编译后的代码中：

_src/parse.js_

```js
ASTCompiler.prototype.escape = function(value) {
  // if (_.isString(value)) {
  //   return '\'' +
  //     value.replace(this.stringEscapeRegex, this.stringEscapeFn) +
  //     '\'';
  } else if (_.isNull(value)) {
    return 'null';
  // } else {
  //   return value;
  // }
};
```