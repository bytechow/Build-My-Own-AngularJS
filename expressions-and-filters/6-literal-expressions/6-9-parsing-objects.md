### 解析对象
#### Parsing Objects

本章最后要介绍的表达式是对象字面量。也就是，像 `{a: 1, b: 2}` 这类的键-值对。在表达式中，对象不仅会被用作数据字面量，也会作为 ngClass 和 ngStyle 这类指令的配置项。

解析对象大体上与解析数组类似，但有几个重要的区别。同样地，我们先来处理空集合的情况。一个空对象应当被解析为一个空对象：

_test/parse_spec.js_

```js
it('will parse an empty object', function() {
  var fn = parse('{}');
  expect(fn()).toEqual({});
});
```

要处理对象，Lexer 需要再支持三个字符 token，分别是标志开始的花括号、标志结束的花括号，以及表示键值对的冒号：

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
  //     (this.ch === '.' && this.isNumber(this.peek()))) {
  //     this.readNumber();
  //   } else if (this.ch === '\'' || this.ch === '"') {
  //     this.readString(this.ch);
    } else if (this.ch === '[' || this.ch === ']' || this.ch === ',' ||
              this.ch === '{' || this.ch === '}' || this.ch === ':') {
  //     this.tokens.push({
  //       text: this.ch
  //     });
  //     this.index++;
  //   } else if (this.isIdent(this.ch)) {
  //     this.readIdent();
  //   } else if (this.isWhitespace(this.ch)) {
  //     this.index++;
  //   } else {
  //     throw 'Unexpected next character: ' + this.ch;
  //   }
  // }
  // return this.tokens;
};
```

这个 `else if` 分支显得有点笨重了。下面我们来为 Lexer 增加一个函数，用于检查当前字符是否满足一系列备选字符中的一个。这个函数会接收一个字符串，然后检查当前字符是否在字符串中存在：

_src/parse.js_

```js
Lexer.prototype.is = function(chs) {
  return chs.indexOf(this.ch) >= 0;
};
```

有了函数，我们就可以让 `lex` 函数的代码更简洁：

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
      (this.is('.') && this.isNumber(this.peek()))) {
      // this.readNumber();
    } else if (this.is('\'"')) {
      // this.readString(this.ch);
    } else if (this.is('[],{}:')) {
  //     this.tokens.push({
  //       text: this.ch
  //     });
  //     this.index++;
  //   } else if (this.isIdent(this.ch)) {
  //     this.readIdent();
  //   } else if (this.isWhitespace(this.ch)) {
  //     this.index++;
  //   } else {
  //     throw 'Unexpected next character: ' + this.ch;
  //   }
  // }
  // return this.tokens;
};
```

对象跟数组一样，也是一个 primary 表达式。`AST.primary` 会查找是否有左花括号，如果发现有，就授权给一个 `object` 方法进行处理：

```js
AST.prototype.primary = function() {
  // if (this.expect('[')) {
  //   return this.arrayDeclaration();
  } else if (this.expect('{')) {
    return this.object();
  // } else if (this.constants.hasOwnProperty(this.tokens[0].text)) {
  //   return this.constants[this.consume().text];
  // } else {
  //   return this.constant();
  // }
};
```