### 一元运算符
#### Unary Operators

我们将从优先级最高的的运算符开始，然后按照优先级递减的顺序向下进行。首先要介绍的是我们已经实现的 primary 表达式：每当有函数被调用或者有计算型或非计算型的属性访问时，它们都会被最先进行计算。而在 primary 表达式之后的是一元运算符。

一元运算符就是只有一个操作数的运算符：

- `-` 会把它的操作数取反，比如 `-42` 或者 `-a`
- `+` 实际上没做什么，但它可以用于强调或让代码更清晰，比如 `+42` 或者 `+a`
- 非运算符会对操作数的布尔值进行取反操作，比如 `!true` 或 `!a`

由于 `+` 运算符最简单，因为下面我们先从它开始讲起，它会原封不动地返回操作数：

_test/parse_spec.js_

```js
it('parses a unary +', function() {
  expect(parse('+42')()).toBe(42);
  expect(parse('+a')({ a: 42 })).toBe(42);
});
```

在 AST 构建器中，我们会加入一个名为 `unary` 的新方法来处理一元元算符，如果当前处理的字符不是一元运算符，它就会回退到使用 `primary` 进行处理：

_src/parse.js_

```js
AST.prototype.unary = function() {
  if (this.expect('+')) {

  } else {
  return this.primary();
  }
};
```

`unary` 方法实际上会构建一个 `UnaryExpression` 的 token，它唯一的参数将会是一个 primary 表达式：

_src/parse.js_

```js
AST.prototype.unary = function() {
  // if (this.expect('+')) {
    return {
      type: AST.UnaryExpression,
      operator: '+',
      argument: this.primary()
    };
  // } else {
  //   return this.primary();
  // }
};
```

`UnaryExpression` 是一个新的 AST 节点类型：

_src/parse.js_

```js
// AST.Program = 'Program';
// AST.Literal = 'Literal';
// AST.ArrayExpression = 'ArrayExpression';
// AST.ObjectExpression = 'ObjectExpression';
// AST.Property = 'Property';
// AST.Identifier = 'Identifier';
// AST.ThisExpression = 'ThisExpression';
// AST.LocalsExpression = 'LocalsExpression';
// AST.MemberExpression = 'MemberExpression';
// AST.CallExpression = 'CallExpression';
// AST.AssignmentExpression = 'AssignmentExpression';
AST.UnaryExpression = 'UnaryExpression';
```

要让一元元算符表达式可以被真正解析，我们还需要在某处地方调用 `unary` 方法。我们会在 `assignment` 中调用 `unary`，也就是我们之前调用 `primary` 的地方。赋值表达式的左右两边不一定是 primary 表达式，也有可能是一元表达式：

_src/parse.js_

```js
AST.prototype.assignment = function() {
  var left = this.unary();
  // if (this.expect('=')) {
    var right = this.unary();
  //   return { type: AST.AssignmentExpression, left: left, right: right };
  // }
  // return left;
};
```

由于 `unary` 可以回退到 `primary`，`assignment` 现在就可以同时兼容它们二者了。

然后我们需要看看 Lexer 需要做些什么事情了。我们的 AST 构建器已经做好处理一元运算符 `+` 的准备了，但 Lexer 还没有发出（emit）这个符号。

在前面的章节中，我们已经处理过几个要发出纯文本字符 token 的情况了，比如 `[, ]` 和 `.`。而对于运算符来说 ，我们需要进行不同的处理：我们需要增加一个“常量”对象 `OPERATORS`，里面会包含所有我们考虑到的运算符：

_src/parse.js_

```js
var OPERATORS = {
  '+': true
};
```

这个对象中的所有属性的值都会是 `true`。我们使用对象而不是数组的原因是，对于常量，对象能让我们更有效滴检查属性是否存在。

Lexer 仍然需要发出 `+`。我们需要在 `lex` 方法中加入最后一个 `else` 分支，这可以让我们在 `OPERATORS` 对象中尝试找到当前字符是否是其中一个运算符：

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
      var op = OPERATORS[this.ch];
      if (op) {
        this.tokens.push({ text: this.ch });
        this.index++;
      } else {
        throw 'Unexpected next character: ' + this.ch;
      }
  //   }
  // }
  
  // return this.tokens;
};
```

基本上，`lex` 现在尝试将该字符与它知道的所有信息进行匹配，如果其他 else 分支都失效的话，就会查看 `OPERATORS` 对象中到底有没有包含这个字符。

最后，我们现在要知道 AST 编译器是如何处理一元运算符的。我们可以返回一个 JavaScript 片段，它由表达式的操作符和对参数的递归求值组成：

_src/parse.js_

```js
case AST.UnaryExpression:
  return ast.operator + '(' + this.recurse(ast.argument) + ')';
```