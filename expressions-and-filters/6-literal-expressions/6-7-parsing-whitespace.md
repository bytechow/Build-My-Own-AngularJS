### 解析空白字符
#### Parsing Whitespace

在我们开始讨论包含多个 token 的表达式之前，先来考虑一下空白字符的问题。像 `‘[1, 2, 3]’`，`‘a = 42’` 还有 `‘aFunction (42)’`，这种表达式都包含了空白符。而它们的共同点就是空白符都可以省略的，也会被解析器忽略掉，Angular 表达式中（几乎）所有的空白符也会这样的。

_test/parse_spec.js_

```js
it('ignores whitespace', function() {
  var fn = parse(' \n42 '); expect(fn()).toEqual(42);
});
```

我们认为空白字符包括空格、回车符、水平和垂直制表符、换行符和非断行空格：

_src/parse.js_

```js
Lexer.prototype.isWhitespace = function(ch) { 
  return ch === ' ' || ch === '\r' || ch === '\t' ||
         ch === '\n' || ch === '\v' || ch === '\u00A0';
};
```

在 `lex` 方法中，如果发现当前字符是上述其中之一的空白字符，只需要将索引指针往前移就行了：

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
  //   } else if (this.isIdent(this.ch)) {
  //     this.readIdent();
    } else if (this.isWhitespace(this.ch)) {
      this.index++;
  //   } else {
  //     throw 'Unexpected next character: ' + this.ch;
  //   }
  // }
  
  // return this.tokens;
};
```