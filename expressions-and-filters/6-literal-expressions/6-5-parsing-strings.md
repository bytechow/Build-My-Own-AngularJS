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

就像 JavaScript 字符串，Angular 表达式字符串中也会有转义字符。我们需要支持的转义符有两种类型：

1. 单字符转义符：换行符 `\n`，换页符 `\f`，回车符 `\r`，水平制表符 `\t`，垂直制表符 `\v`，单引号 `\'`，还有双引号 `\"`。
2. Unicode 转义序列：这是以 `\u` 开头，后面跟着 4 个十六进制的字符代码。举个例子，`\u00A0` 表示一个不间断空格字符。

我们先来看看如何处理单字符转义符。首先，我们应该能解析包含引号的字符串：

_test/parse_spec.js_

```js
it('can parse a string with single quotes inside', function() {
  var fn = parse("'a\\\'b'");
  expect(fn()).toEqual('a\'b');
});

it('can parse a string with double quotes inside', function() {
  var fn = parse('"a\\\"b"');
  expect(fn()).toEqual('a\"b');
});
```

我们要做的就是在解析过程中，一旦发现反斜杠 `\`，就进入“转义模式”，这样我们就可以对接下来的字符做特殊处理：

_src/parse.js_

```js
Lexer.prototype.readString = function(quote) {
  // this.index++;
  // var string = '';
  var escape = false;
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
    if (escape) {
      
    } else if (ch === quote) {
      // this.index++;
      // this.tokens.push({
      //   text: string,
      //   value: string
      // });
      // return;
    } else if (ch === '\\') {
      escape = true;
  //   } else {
  //     string += ch;
  //   }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```

进入转义模式以后，如果我们发现有单字符转义符，就要看一下它属于哪种单字符转义符，然后替换成对应的转义字符。我们会把要支持的转义字符都放到 `parse.js` 顶层作用域的一个对象常量中进行保存。它会包含我们上面单元测试中出现的引号字符：

_src/parse.js_

```js
var ESCAPES = {'n':'\n', 'f':'\f', 'r':'\r', 't':'\t',
               'v':'\v', '\'':'\'', '"':'"'};
```

然后在 `readString` 里，我们会从这个对象中查找是否有这个转义字符。如果有，我们就会替换成对应的转义字符，再把替换后的字符加入到结果字符穿中。如果没有找到，我们就把这个字符原封不动地加入到结果字符串中，直接忽略掉用于转义的反斜杠就好：

_src/parse.js_

```js
Lexer.prototype.readString = function(quote) {
  // this.index++;
  // var string = '';
  // var escape = false;
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
  //   if (escape) {
      var replacement = ESCAPES[ch];
      if (replacement) {
        string += replacement;
      } else {
        string += ch;
      }
      escape = false;
  //   } else if (ch === quote) {
  //     this.index++;
  //     this.tokens.push({
  //       text: string,
  //       value: string
  //     });
  //     return;
  //   } else if (ch === '\\') {
  //     escape = true;
  //   } else {
  //     string += ch;
  //   }
  //   this.index++;
  // }
  // throw 'Unmatched quote';
};
```

在进入 AST 编译阶段之前，我们还需要解决几个问题。当 AST 编译器遇到像 `'` 和 `"` 这样的字面量时，它只会直接把它放到结果中，这样会产出一些非访的 JavaScript 代码。编译器的 `escape` 方法需要能够处理这些字符。我们可以在转义过程中加入一个正则进行转义：

_src/parse.js_

```js
ASTCompiler.prototype.escape = function(value) {
  // if (_.isString(value)) {
    return '\'' +
      value.replace(this.stringEscapeRegex, this.stringEscapeFn) +
      '\'';
  // } else {
  //   return value;
  // }
};
```

需要进行转义，是除了空格、字母和数字以外的字符：

_src/parse.js_

```js
ASTCompiler.prototype.stringEscapeRegex = /[^ a-zA-Z0-9]/g;
```

而在用于替换的函数中，我们会获取要转义字符的 Unicode 数字代码（利用 [charCodeAt](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/String/charCodeAt)），然后把它转成相对应的十六进制值（以 16 为基数）的 Unicode 转义序列，然后我们就可以安全地把这哥序列加入到要生成的 JavaScript 代码中：

_src/parse.js_

```js
ASTCompiler.prototype.stringEscapeFn = function(c) {
  return '\\u' + ('0000' + c.charCodeAt(0).toString(16)).slice(-4);
};
```

最后，我们要考虑一下输入表达式本身含有转义序列的情况：

_test/parse_test.js_

```js
it('will parse a string with unicode escapes', function() {
  var fn = parse('"\\u00A0"');
  expect(fn()).toEqual('\u00A0');
});
````