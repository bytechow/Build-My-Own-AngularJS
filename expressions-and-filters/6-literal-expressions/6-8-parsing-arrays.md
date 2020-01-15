### 解析数组
#### Parsing Array

数字，字符串，布尔值和 `null` 都是所谓的 _标量_ 字面表达式。它们就是简单的、单个的值，每个值只包含一个标记。接下来，我们会把注意力转向包含多个 token 的表达式。第一个要处理的类型是数组。

最简单的数组就是一个空数组。空数组由一个左方括号和右方括号组成：

_test/parse_spec.js_

```js
it('will parse an empty array', function() { 
  var fn = parse('[]');
  expect(fn()).toEqual([]);
});
```

虽然空数组很简单，但它也是我们遇到的第一个包含多个 token 的表达式。Lexer 会根据表达式输出两个 token，每个方括号一个。我们会在 `lex` 函数输出这两个 token：

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
    } else if (this.ch === '[' || this.ch === ']') {
      this.tokens.push({
        text: this.ch
      });
      this.index++;
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

在 AST builder 中，我们也需要考虑怎么处理出现多个 token 的情况了。我们现在有两个 token ——`[` 和 `]`，这在 AST 应该表示一个数组节点。

就像常量一样，数组是一个基本表达式（primary expressions），所以我们会在 `AST.primary` 方法中处理数组。一个基本表达式如果是以左方括号作为开头，我们就把它当作一个数组的声明：

_src/parse.js_

```js
AST.prototype.primary = function() {
  if (this.expect('[')) {
    return this.arrayDeclaration();
  } else if (this.constants.hasOwnProperty(this.tokens[0].text)) {
  //   return this.constants[this.tokens[0].text];
  // } else {
  //   return this.constant();
  // }
};
```

这样要用到的 `expect` 函数我们还没有实现。它与目前需要处理多 token 的实际情况有关。`expect` 方法会检查下一个 token 是否是我们想要的，如果是，就返回这个 token。它同时会把这个 token 从 `this.tokens` 中移除掉，这样我们就能“前进”到下一个 token 了：

_src/parse.js_

```js
AST.prototype.expect = function(e) { 
  if (this.tokens.length > 0) {
    if (this.tokens[0].text === e || !e) {
      return this.tokens.shift(); 
    }
  }
};
```