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

`object` 方法的结构跟 `arrayDeclaration` 基本一致。它会对对象字面量进行消费，包括右方括号，然后返回一个类型为 `ObjectExpression` 的 AST 节点：

_src/parse.js_

```js
AST.prototype.object = function() { 
  this.consume('}');
  return {type: AST.ObjectExpression};
};
```

同样地，这个类型也要先定义好：

_src/parse.js_

```js
AST.Program = 'Program';
AST.Literal = 'Literal';
AST.ArrayExpression = 'ArrayExpression';
AST.ObjectExpression = 'ObjectExpression';
```

而 AST 编译器的职责就是，如果在 `recurse` 中发现 `ObjectExpression` 节点，就返回一个对象字面量：

_src/parse.js_

```js
case AST.ObjectExpression:
  return '{}';
```

当对象非空时，它的键就会是一个标志符（identifiers）或字符串，它的值也可能会是另一个表达式。下面是一个 key 为字符串的测试用例：

_test/parse_spec.js_

```js
it('will parse a non-empty object', function() {
  var fn = parse('{"a key": 1, \'another-key\': 2}');
  expect(fn()).toEqual({ 'a key': 1, 'another-key': 2 });
});
```

就像构建数组的 AST 时一样，构建对象的 AST 也会用到 `do...while` 循环来对以逗号进行分隔的键（keys）和值（values）进行消耗：

_src/parse.js_

```js
AST.prototype.object = function() {
  if (!this.peek('}')) {
    do {} while (this.expect(','));
  }
  // this.consume('}');
  // return { type: AST.ObjectExpression };
};
```

在循环体中，我们首先要把要消耗的常量字符读取进来。在此基础上，我们构建了另一个名为 `Property` 的 AST 节点：

_src/parse.js_

```js
AST.prototype.object = function() {
  // if (!this.peek('}')) {
  //   do {
      var property = { type: AST.Property };
      property.key = this.constant();
  //   } while (this.expect(','));
  // }
  // this.consume('}');
  // return { type: AST.ObjectExpression };
};
```

当然，我们也要对这种类型进行声明：

_src/parse.js_

```js
// AST.Program = 'Program';
// AST.Literal = 'Literal';
// AST.ArrayExpression = 'ArrayExpression'; AST.ObjectExpression = 'ObjectExpression';
AST.Property = 'Property';
```

然后，我们会对分隔键值的冒号字符进行消耗：

_src/parse.js_

```js
AST.prototype.object = function() {
  // if (!this.peek('}')) {
  //   do {
  //     var property = { type: AST.Property };
  //     property.key = this.constant();
      this.consume(':');
  //   } while (this.expect(','));
  // }
  // this.consume('}');
  // return { type: AST.ObjectExpression };
};
```

最后我们需要对值进行消耗，我们需要把这个值看作另一个 primary AST 节点，然后添加到 property 节点中去：

_src/parse.js_

```js
AST.prototype.object = function() {
  // if (!this.peek('}')) {
  //   do {
  //     var property = { type: AST.Property };
  //     property.key = this.constant();
  //     this.consume(':');
      property.value = this.primary();
  //   } while (this.expect(','));
  // }
  // this.consume('}');
  // return { type: AST.ObjectExpression };
};
```

然后，我们会把循环中生成的属性收集起来，然后把它们都放到 `ObjectExpression` 节点上：

_src/parse.js_

```js
AST.prototype.object = function() {
  var properties = [];
  // if (!this.peek('}')) {
  //   do {
  //     var property = { type: AST.Property };
  //     property.key = this.constant();
  //     this.consume(':');
  //     property.value = this.primary();
      properties.push(property);
  //   } while (this.expect(','));
  // }
  // this.consume('}');
  return { type: AST.ObjectExpression, properties: properties };
};
```

编译时，我们会为每个属性生成对应的 JavaScript 代码，然后把它放到要生成的对象字面量中去：

_src/parse.js_

```js
// case AST.ObjectExpression:
  var properties = _.map(ast.properties, _.bind(function(property) {
    
  }, this));
  return '{' + properties.join(',') + '}';
```

键是一个 `Constant` 节点，它的 `value` 是一个字符串，我们需要对这个字符串进行转义：

_src/parse.js_

```js
// case AST.ObjectExpression:
//   var properties = _.map(ast.properties, _.bind(function(property) {
    var key = this.escape(property.key.value);
    
//   }, this));
// return '{' + properties.join(',') + '}';
```

而属性值可能是任何类型的表达式，我们可以直接使用 `recurse` 进行处理。我们会在生成的键和值之间使用冒号进行分隔：

_src/parse.js_

```js
case AST.ObjectExpression:
  // var properties = _.map(ast.properties, _.bind(function(property) {
  //   var key = this.escape(property.key.value);
    var value = this.recurse(property.value);
    return key + ':' + value;
//   }, this));
// return '{' + properties.join(',') + '}';
```

对象的键也并不总是字符串。它们也可能是（本身）没有引号的标志符：

_test/parse_spec.js_

```js
it('will parse an object with identifier keys', function() {
  var fn = parse('{a: 1, b: [2, 3], c: {d: 4}}');
  expect(fn()).toEqual({ a: 1, b: [2, 3], c: { d: 4 } });
});
```

这个单元测试没有通过，是因为 AST 消耗的对象键都是由 `readIdent` 生成的标志符 token，但实际上我们希望它们是常量。下面，我们先对 `readIndent` 做一点修改，以便对标识符 token 进行标记：

_src/parse.js_

```js
Lexer.prototype.readIdent = function() {
  // var text = '';
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index);
  //   if (this.isIdent(ch) || this.isNumber(ch)) {
  //     text += ch;
  //   } else {
  //     break;
  //   }
  //   this.index++;
  // }
  // var token = {
  //   text: text,
    identifier: true
  // };
  // this.tokens.push(token);
};
```

AST 构建器需要检查当前 token 是否带有这个标记，如果有则组成一个_标识符_节点，否则构建一个常量节点就好了：

标识符节点是一种新节点，它的类型是 `Identifier`。这种节点的 `name` 属性值就是标识符 token 的文本值：

_src/parse.js_

```js
AST.prototype.identifier = function() {
  return {type: AST.Identifier, name: this.consume().text};
};
```

我们需要引入一个叫 `Identifier` 的类型：

_src/parse.js_

```js
// AST.Program = 'Program';
// AST.Literal = 'Literal';
// AST.ArrayExpression = 'ArrayExpression';
// AST.ObjectExpression = 'ObjectExpression';
// AST.Property = 'Property';
AST.Identifier = 'Identifier';
```

后面我们还会在 AST 里的其他地方用到标识符节点，但现在它们仅存在于对象的键中。

现在我们需要在 AST 编译器中检查对象的键是否是一个 `Identifier` 节点。这会影响到我们要用节点的哪个属性来访问实际要生成的键：

_src/parse.js_

```js
case AST.ObjectExpression:
  // var properties = _.map(ast.properties, _.bind(function(property) {
    var key = property.key.type === AST.Identifier ?
      property.key.name :
      this.escape(property.key.value);
//     var value = this.recurse(property.value);
//     return key + ':' + value;
//   }, this));
// return '{' + properties.join(',') + '}';
```

现在，我们已经实现了对 Angular 表达式语言中所有类型的字面量的解析！