### 关系和相等运算符
#### Relational And Equality Operators

在算术运算符之后，优先级较高的是各种用于比较的运算符。对数字来说，有四种关系运算符：

_test/parse_spec.js_

```js
it('parses relational operators', function() {
  expect(parse('1 < 2')()).toBe(true);
  expect(parse('1 > 2')()).toBe(false);
  expect(parse('1 <= 2')()).toBe(true);
  expect(parse('2 <= 2')()).toBe(true);
  expect(parse('1 >= 2')()).toBe(false);
  expect(parse('2 >= 2')()).toBe(true);
});
```

无论对于数字还是其他类型的值来说，都会有相等性校验以及非相等性校验。Angular 表达式同时支持 JavaScript 中的严格和松散两种模式的相等性比较：

_test/parse_spec.js_

```js
it('parses equality operators', function() {
  expect(parse('42 == 42')()).toBe(true);
  expect(parse('42 == "42"')()).toBe(true);
  expect(parse('42 != 42')()).toBe(false);
  expect(parse('42 === 42')()).toBe(true);
  expect(parse('42 === "42"')()).toBe(false);
  expect(parse('42 !== 42')()).toBe(false);
});
```

在这两个操作符家族中，关系运算符的优先级更高：

_test/parse_spec.js_

```js
it('parses relationals on a higher precedence than equality', function() {
  expect(parse('2 == "2" > 2 === "2"')()).toBe(false);
});
```

这个测试的执行顺序是这样的：

1. 2 == “2” > 2 === “2”
2. 2 == false === “2”
3. false === “2”
4. false

而不是：

1. 2 == “2” > 2 === “2”
2. true > false
3. 1 > 0
4. true

> [ECMAScript 规范](http://www.ecma-international.org/ecma-262/5.1/%23sec-9.3)，当 `true` 和 `false ` 出现在数字运算时，`true` 会被视作 `1`，而 `false` 会被视作 `0`，这像上面的第三步那样。

关系运算符和相等性运算符的优先级都低于加法类运算符：

_test/parse_spec.js_

```js
it('parses additives on a higher precedence than relationals', function() {
  expect(parse('2 + 3 < 6 - 2')()).toBe(false);
});
```

这个测试可以检验执行顺序会像下面这样：

1. 2+3<6- 2
2. 5 < 4
3. false

而不是：


1. 2+3<6- 2
2. 2 + true - 2
3. 2+1- 2
4. 1

这八个新运算符都会被加入到 `OPERATORS` 对象中去：

_src/parse.js_

```js
var OPERATORS = {
  '+': true,
  '-': true,
  '!': true,
  '*': true,
  '/': true,
  '%': true,
  '==': true,
  '!=': true,
  '===': true,
  '!==': true,
  '<': true,
  '>': true,
  '<=': true,
  '>=': true
};
```

在 AST 构建器中，我们需要引入两个新函数——一个用于处理相等运算符，另一个用于处理关系型运算符。我们不能使用同一个函数来同时处理这两类运算符，否则会破坏我们的优先级规则。这两个函数有相似的构成：

_src/parse.js_

```js
AST.prototype.equality = function() {
  var left = this.relational();
  var token;
  while ((token = this.expect('==', '!=', '===', '!=='))) {
    left = {
      type: AST.BinaryExpression,
      left: left,
      operator: token.text,
      right: this.relational()
    };
  }
  return left;
};
AST.prototype.relational = function() {
  var left = this.additive();
  var token;
  while ((token = this.expect('<', '>', '<=', '>='))) {
    left = {
      type: AST.BinaryExpression,
      left: left,
      operator: token.text,
      right: this.additive()
    };
  }
  return left;
};
```

现在，相等运算时除了赋值运算以外最低优先级的运算，因此在处理赋值运算的函数中，我们需要委托给 `equality` 函数：

_src/parse.js_

```js
AST.prototype.assignment = function() {
  var left = this.equality();
  if (this.expect('=')) {
    var right = this.equality();
    return { type: AST.AssignmentExpression, left: left, right: right };
  }
  return left;
};
```

要支持这些函数，我们还需要在 `Lexer.lex` 中做些改变。在本章开头，我们已经加入了一个条件分支来通过 `OPERATORS` 对象查找运算符。但目前在这个对象中的运算符都只包含一个字符而已。现在我需要需要处理的运算符有两个字符，比如 `==`，或甚至三个字符，比如 `===`。因此在这个条件分支，我们需要支持这些多字符运算符。它会首先看看后面三个字符是否能匹配到第一个运算符，然后看第一个字符和第二个字符组合后是否匹配到一个运算符，然后看看这三个字符的组合是否匹配到一个运算符：

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
  //     (this.is('.') && this.isNumber(this.peek()))) {
  //     this.readNumber();
  //   } else if (this.is('\'"')) {
  //     this.readString(this.ch);
  //   } else if (this.is('[],{}:.()=')) {
  //     this.tokens.push({
  //       text: this.ch
  //     });
  //     this.index++;
  //   } else if (this.isIdent(this.ch)) {
  //     this.readIdent();
  //   } else if (this.isWhitespace(this.ch)) {
  //     this.index++;
  //   } else {
      var ch = this.ch;
      var ch2 = this.ch + this.peek();
      var ch3 = this.ch + this.peek() + this.peek(2);
      var op = OPERATORS[ch];
      var op2 = OPERATORS[ch2];
      var op3 = OPERATORS[ch3];
      if (op || op2 || op3) {
        var token = op3 ? ch3 : (op2 ? ch2 : ch);
        this.tokens.push({ text: token });
        this.index += token.length;
  //     } else {
  //       throw 'Unexpected next character: ' + this.ch;
  //     }
  //   }
  // }
  //
  // return this.tokens;
};
```

我需要对这段代码使用到的 `Lexer.peek` 方法进行修改，让它不仅能查看后面一个字符，还能指定把从当前索引开始后多少位的字符串都截取出来。它会接收一个可选参数 `n`，可选参数的默认值为 `1`：

_src/parse.js_

```js
Lexer.prototype.peek = function(n) {
  n = n || 1;
  return this.index + n < this.text.length ?
    this.text.charAt(this.index + n) :
    false;
};
```

但现在相等性运算符的测试还未能通过，尽管我们看上去已经准备好了一切。问题是，在上一个章节中我们把等号 `=` 看作是一个文本 token，以便用于处理赋值表达式。而现在，当 Lexer 看到 `==` 中的第一个 `=` 时，就会弹出它，而不管它是否是其他运算符的一部分。

我们要做的首先是把 `=` 从文本 Token 集合中移除掉，也就是从：

```js
} else if (this.is('[],{}:.()=')) {
```

变成：

```js
} else if (this.is('[],{}:.()')) {
```

然后，我们需要往运算符集合中加入单个的等号运算符：

_src/parse.js_

```js
var OPERATORS = {
  // '+': true,
  // '-': true,
  // '!': true,
  // '*': true,
  // '/': true,
  // '%': true,
  '=': true,
  // '==': true,
  // '!=': true,
  // '===': true,
  // '!==': true,
  // '<': true,
  // '>': true,
  // '<=': true,
  // '>=': true
};
```

现在等号会被作为一个运算符 token 弹出，而不是文本 token。它仍然会被构建到一个赋值节点中去，由于这两种 token 都会有一个值为 `=` 的文本属性，这是 AST 构建器感兴趣的地方。