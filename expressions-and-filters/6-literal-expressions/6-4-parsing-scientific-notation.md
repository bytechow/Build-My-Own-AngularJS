### 解析科学记数法
#### Parsing Scientific Notation

在 Angular 表达式中，第三种也是最后一种表示数字的方法是科学记数法，它由两个数字构成：系数（coefficient）和指数（exponent），它们之间用字符 `e` 分割。例如，数字 `42,000` 或者 `42 * 10³`，都可以表示为 `42e3`。下面是对应的单元测试：

_test/parse_spec.js_

```js
it('can parse a number in scientific notation', function() {
  var fn = parse('42e3');
  expect(fn()).toBe(42000);
});
```

另外，科学记数法中的系数也不一定就是一个整数：

_test/parse_spec.js_

```js
it('can parse scientific notation with a float coefficient', function() {
  var fn = parse('.42e2');
  expect(fn()).toBe(42);
});
```

而科学记数法中的指数也可能是负数，让系数乘以负的 10 次方：

_test/parse_spec.js_

```js
it('can parse scientific notation with negative exponents', function() {
  var fn = parse('4200e-2');
  expect(fn()).toBe(42);
});
```

指数也能被显式地标识为一个正数，只要在前面加一个 `+` 号就好了：

_test/parse_spec.js_

```js
it('can parse scientific notation with the + sign', function() {
  var fn = parse('.42e+2');
  expect(fn()).toBe(42);
});
```

最后，系数与指数之间的分隔符也可能是大写的 `E`：

_test/parse_spec.js_

```js
it('can parse upper case scientific notation', function() {
  var fn = parse('.42E2');
  expect(fn()).toBe(42);
});
```

我们已经把所有科学记数法的规则都定义好了，又该如何实现它呢？最直接的方法就是先把字符变成小写，然后检查字符是否 `e`、`-` 或 `+`，是的话就把这个字符传递下去，最后依靠 JavaScript 的数字类型机制完成剩余的工作。这确实能让我们的单元测试通过：

```js
Lexer.prototype.readNumber = function() {
  // var number = '';
  // while (this.index < this.text.length) {
  var ch = this.text.charAt(this.index).toLowerCase();
  if (ch === '.' || ch === 'e' || ch === '-' ||
      ch === '+' || this.isNumber(ch)) {
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

你可能也猜到，事情并没有那么简单。虽然目前的代码能够正确处理科学记数法，但它对非法符号太“仁慈”了，会让类似下边的残缺数字字面量也能通过校验：

_test/parse_spec.js_

```js
it('will not parse invalid scientific notation', function() {
  expect(function() { parse('42e-'); }).toThrow();
  expect(function() { parse('42e-a'); }).toThrow();
});
```

下面我们来严格规范一下。首先，我们需要引入_指数运算符_（exponent operator）这个概念。也就是说，允许出现在科学记数法 `e` 字符后的字符。可能是个数字、加号或减号：

_src/parse.js_

```js
Lexer.prototype.isExpOperator = function(ch) {
  return ch === '-' || ch === '+' || this.isNumber(ch);
};
```

接下来，我们需要在 `readNumber` 中使用这个校验。我们需要先撤销之前的修改，然后加入一个 `else` 分支来处理科学记数法：

_src/parse.js_

```js
Lexer.prototype.readNumber = function() {
  // var number = '';
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index).toLowerCase();
    if (ch === '.' || this.isNumber(ch)) {
      // number += ch;
    } else {
      
    }
  //   this.index++;
  // }
  // this.tokens.push({
  //   text: number,
  //   value: Number(number)
  // });
};
```

我们需要考虑以下三个情况：

- 如果当前字符为 `e`，且下一个字符是一个合法的指数运算符，那就把这个字符加入到结果中，然后继续读取下一个字符。
- 如果当前字符为 `+` 或 `-`，前一个字符是 `e`，后一个字符是一个数字，就把这个字符加入到结果中，然后继续读取。
- 如果当前字符为 `+` 或 `-`，前一个字符是 `e`，而后一个字符不是数字，我们就要抛出异常了。
- 除了以上三种情况之外，我们都会结束对数字的读取，然后返回 token 结果。

下面是对应的代码：

_src/parse.js_

```js
Lexer.prototype.readNumber = function() {
  // var number = '';
  // while (this.index < this.text.length) {
  //   var ch = this.text.charAt(this.index).toLowerCase();
  //   if (ch === '.' || this.isNumber(ch)) {
  //     number += ch;
  //   } else {
      var nextCh = this.peek();
      var prevCh = number.charAt(number.length - 1);
      if (ch === 'e' && this.isExpOperator(nextCh)) {
        number += ch;
      } else if (this.isExpOperator(ch) && prevCh === 'e' &&
        nextCh && this.isNumber(nextCh)) {
        number += ch;
      } else if (this.isExpOperator(ch) && prevCh === 'e' &&
        (!nextCh || !this.isNumber(nextCh))) {
        throw 'Invalid exponent';
      } else {
        break;
      }
  //   }
  //   this.index++;
  // }
  // this.tokens.push({
  //   text: number,
  //   value: Number(number)
  // });
};
```

注意，在第二、第三个条件分支，我们重用了 `isExpOperator` 函数来检查当前字符是否是 `+` 和 `-`。`isExpOperator` 也能接受数字，但由于数字会被 `while` 循环的第一个 `if` 条件分支捕获处理，所以这里不可能出现数字。

这个函数相当复杂，但它让我们能够在 Angular 表达式中处理几乎所有数字类型——除了负数，我们会在支持 `-` 运算符的时候再处理负数。