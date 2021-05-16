### 解析浮点数
#### Parsing Floating Point Numbers

我们的 Lexer（词法分析器）现在只能处理整数，但像 `4.2` 这种浮点数就无法处理了。

_test/parse_spec.js_

```js
it('can parse a floating point number', function() {
  var fn = parse('4.2');
  expect(fn()).toBe(4.2);
});
```

要解决这个问题也很简单，我们只需要调整一下 `readNumber` 就可以，让它不仅可以接受数字，也可以接受点符号（dot）：

```js
Lexer.prototype.readNumber = function() {
  // var number = '';
  // while (this.index < this.text.length) {
    // var ch = this.text.charAt(this.index);
    if (ch === '.' || this.isNumber(ch)) {
  //     number += ch;
  //   } else {
  //     break;
  //   }
  //   this.index++;
  // }
  // this.tokens.push({
  //   text: number,
  //   value: Number(number)
  // });
};
```

我们不需要对点符号进行特殊处理，因为原生的 JavaScript 本来就支持这种数字类型。

当一个浮点数的整数部分是 0 的时候，Angular 表达式允许你忽略不写整数值，这跟 JavaScript 的规则是一致的。但目前我们的代码还不支持这种形式，所以下面这个单元测试会报错：

_test/parse_spec.js_

```js
it('can parse a floating point number without an integer part', function() {
  var fn = parse('.42');
  expect(fn()).toBe(0.42);
});
```

原因在于 lex 函数中，我们使用 `readNumber` 读取数字时判断条件仅仅是看当前字符是不是一个数字。当当前字符是一个点（dot），而且下一个字符是一个数字的话，我们也要判断这种情况符合读取数字的规则。

首先，为了判断下一个字符的类型，我们会在 lexer 中新增一个叫 `peek` 的函数。它会返回在文本中的下一个字符，但不会增加字符索引。如果没有下一个字符了，`peek` 返回的是 `false`：

_src/parse.js_

```js
Lexer.prototype.peek = function() {
  return this.index < this.text.length - 1 ?
    this.text.charAt(this.index + 1) :
    false; 
};
```

`lex` 函数会使用这个函数的返回值作为是否继续读取数字的判断条件：

_src/parse.js_

```js
Lexer.prototype.lex = function(text) {
  // this.text = text;
  // this.index = 0;
  // this.ch = undefined;
  // this.tokens = [];

  // while (this.index < this.text.length) {
  //   this.ch = this.text.charAt(this.index);
    if (this.isNumber(this.ch) ||
         (this.ch === '.' && this.isNumber(this.peek()))) {
      this.readNumber();
  //   } else {
  //     throw 'Unexpected next character: '+this.ch;
  //   }
  // }
  // return this.tokens;
};
```

这样我们就能处理浮点数了！