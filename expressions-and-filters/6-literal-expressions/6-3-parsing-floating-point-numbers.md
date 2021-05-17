### 解析浮点数
#### Parsing Floating Point Numbers

我们的 Lexer（词法分析器）现在能处理整数了，但还无法处理像 `4.2` 这样的浮点数。

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

我们不需要为了解析点符号而做什么特殊处理，因为原生 JavaScript 内置的强制数字转换可以处理它。

当浮点数的整数部分为 0 时，Angular 表达式允许我们忽略不写整数值，这跟 JavaScript 是一样的。但目前我们的代码还不支持这种形式，所以下面这个单元测试会报错：

_test/parse_spec.js_

```js
it('can parse a floating point number without an integer part', function() {
  var fn = parse('.42');
  expect(fn()).toBe(0.42);
});
```

原因是现在在 lex 函数中，我们仅仅是通过检查当前字符是否是数字来决定是否执行 `readNumber` 读取数字。当当前字符是一个点时（dot）且下一个字符是数字的话，我们也应该执行 `readNumber` 读取数字。

首先为了获取下一个字符的类型，我们在 lexer 中新增一个叫 `peek` 的函数。它会返回字符串中的下一个字符，但不会递增字符索引。如果没有下一个字符了，`peek` 会返回 `false`：

_src/parse.js_

```js
Lexer.prototype.peek = function() {
  return this.index < this.text.length - 1 ?
    this.text.charAt(this.index + 1) :
    false; 
};
```

`lex` 函数会根据这个函数的返回值来判断是否要继续读取数字：

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

这就是解析浮点数的实现过程了！