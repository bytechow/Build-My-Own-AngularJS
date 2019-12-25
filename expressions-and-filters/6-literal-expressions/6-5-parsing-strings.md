### 解析字符串
#### Parsing Strings

处理完数字，接下来我们来处理一下字符串。要让语法分析器能够处理字符串，其实基本上跟处理数字一样简单，但还是要处理几种特殊情况。

简单来说，表达式中的字符串就是用单引号或双引号括起来的字符序列：

_test/parse_spec.js_

```js
it('can parse a string in single quotes', function() {
  var fn = parse("'abc'");
  expect(fn()).toEqual('abc');
});

it('can parse a string in double quotes', function() {
  var fn = parse('"abc"');
  expect(fn()).toEqual('abc');
});
```

在 `lex` 函数中我们可以判断当前字符是不是这些引号中的一个，若是的话，我们就调用一个用于读取字符串的函数，稍后我们会实现这个函数：

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
    } else if (this.ch === '\'' || this.ch === '"') {
      this.readString();
  //   } else {
  //     throw 'Unexpected next character: ' + this.ch;
  //   }
  // }
  // return this.tokens;
};
```
 
从顶层结构上来说，`readString` 函数跟 `readNumber` 很类似，它会使用一个 `while` 循环来逐个读取表达式中的字符，从而构建出一个字符串，并把它放到一个本地变量中。与 `readNumber` 的一个重要区别是，在进入 `while` 循环之前，我们会将字符索引自增 1，让它能跳过开头的引号字符：

_src/parse.js_

```js
Lexer.prototype.readString = function() {
  this.index++;
  var string = '';
  while (this.index < this.text.length) {
    var ch = this.text.charAt(this.index);

    this.index++;
  } 
};
```