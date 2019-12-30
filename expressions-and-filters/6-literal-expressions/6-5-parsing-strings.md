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

那我们在循环语句中要做些什么呢？有两件事：如果当前字符不是引号，我们就把它拼接到结果字符串中去。如果是引号，说明该字符串已经结束了，我们要把当前拼接好的字符串加入到 token 数组中去，然后结束循环。在循环结束以后如果依然处于读取到字符串的阶段，我们就会抛出一个异常，因为这代表着表达式结束了这个字符串还未结束：

_src/parse.js_

```js
Lexer.prototype.readString = function() {
  // this.index++;
  // var string = '';
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
    if (ch === '\'' || ch === '"') {
      this.index++;
      this.tokens.push({
        text: string,
        value: string
      });
      return;
    } else
      {
      string += ch;
    }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```

这对于解析字符串是一个良好的开始，但是我们还没有完成。单元测试还未能通过，这是因为目前这个 token 在 AST 处理结束时会作为一个字面量输出，它的值会原封不动地编译到最终的 JavaScript 函数中去。表达式 `'abc'` 生成的函数就像下面这样：

```js
function() {
  return abc;
}
```

我们可以发现，原本字符串两侧应该出现引号都不见了，因此函数转而会去尝试查找名称为 `abc` 的变量！

我们的 AST 编译器需要能够_转义_字符串，以便让它们能在 JavaScript 函数中加上应有的引号。我们会使用一个名为 `escape` 的方法来干这个事情：

_src/parse.js_

```js
case AST.Literal:
  return this.escape(ast.value);
```

这个方法可以在一个变量的两侧加上引号，但这个变量必须是字符串：

_src/parse.js_

```js
ASTCompiler.prototype.escape = function(value) {
  if (_.isString(value)) {
    return '\'' + value + '\'';
  } else {
    return value;
  }
};
```

既然我们使用了 `_.isString`，我们就得在 parse.js 中引入 LoDash 了：

_src/parse.js_

```js
'use strict';

var _ = require('lodash');
```

我们对于输入字符串的开始和结束的标志也太“宽容”了，因为我们现在允许使用与开始标识不同的引号来标记字符串的结束：

_test/parse_spec.js_

```js
it('will not parse a string with mismatching quotes', function() {
  expect(function() { parse('"abc\''); }).toThrow();
});
```

我们需要保证标记字符串开始和结束的引号要一致才行。首先在 `lex` 函数中在调用 `readString` 方法时，我们需要把标记字符串开始的引号传递过去：

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
      this.readString(this.ch);
  //   } else {
  //     throw 'Unexpected next character: ' + this.ch;
  //   }
  // }
  // return this.tokens;
};
```

现在在 `readString` 中，我们就可以直接用这个传入的引号字符来判断字符串是否结束了，而不再使用字面量 `'` 或 `"`：

_src/parse.js_

```js
Lexer.prototype.readString = function(quote) {
  // this.index++;
  // var string = '';
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
    if (ch === quote) {
      // this.index++;
      // this.tokens.push({
      //   text: string,
      //   value: string
      // });
      // return;
  //   } else {
  //     string += ch;
  //   }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```