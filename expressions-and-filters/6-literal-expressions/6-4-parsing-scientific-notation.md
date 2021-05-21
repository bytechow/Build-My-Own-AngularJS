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

另外，科学记数法中的系数不一定是整数：

_test/parse_spec.js_

```js
it('can parse scientific notation with a float coefficient', function() {
  var fn = parse('.42e2');
  expect(fn()).toBe(42);
});
```

科学记数法的指数也可以是负数，结果就是系数乘以负的 10 次方：

_test/parse_spec.js_

```js
it('can parse scientific notation with negative exponents', function() {
  var fn = parse('4200e-2');
  expect(fn()).toBe(42);
});
```

指数也可以显式地表示为正数，只要在它前面加一个 `+` 号就好了：

_test/parse_spec.js_

```js
it('can parse scientific notation with the + sign', function() {
  var fn = parse('.42e+2');
  expect(fn()).toBe(42);
});
```

最后，系数与指数之间的分隔符也可以是大写的 `E`：

_test/parse_spec.js_

```js
it('can parse upper case scientific notation', function() {
  var fn = parse('.42E2');
  expect(fn()).toBe(42);
});
```

有了科学记数法的规范以后，我们又该怎么实现它呢？最直接的方法就是先把每个字符变成小写，如果字符是 `e`、`-` 或 `+`，我们就直接把它当作数字的一部分拼接起来，最后利用 JavaScript 的数字强制转换机制完成剩下的工作。这样确实就能让我们的单元测试通过：

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

你可能也猜到了，事情并没有那么简单。虽然目前的代码能够正确解析科学记数法，但它对无效的计数法太宽松了，以至于连下边这种残缺数字字面量都能通过校验：

_test/parse_spec.js_

```js
it('will not parse invalid scientific notation', function() {
  expect(function() { parse('42e-'); }).toThrow();
  expect(function() { parse('42e-a'); }).toThrow();
});
```

要把事情弄清楚，就需要先明确_指数运算符_（exponent operator）的概念。指数运算符是指在科学记数法中允许出现在 `e` 字符后的字符。它可能是一个数字、加号或减号：

_src/parse.js_

```js
Lexer.prototype.isExpOperator = function(ch) {
  return ch === '-' || ch === '+' || this.isNumber(ch);
};
```

接下来，我们需要在 `readNumber` 方法中使用这个校验。首先，我们需要撤销之前的修改，然后加入一个空的 `else` 分支，我们将在这里处理科学记数法：

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

我们需要考虑三种情况：

- 如果当前字符为 `e`，而它的下一个字符是一个合法的指数运算符，那就把这个字符加入到结果中，然后继续读取下一个字符。
- 如果当前字符为 `+` 或 `-`，前一个字符是 `e`，后一个字符是一个数字，就把这个字符加入到结果中，然后继续读取。
- 如果当前字符为 `+` 或 `-`，前一个字符是 `e`，而后一个字符不是数字，我们就要抛出异常了。
- 否则，我们会终止读取数字，然后返回 token 结果。

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

注意，在第二个和第三个条件分支中，我们通过重用 `isExpOperator` 函数来检查当前字符是不是 `+` 和 `-`。虽然 `isExpOperator` 也接受数字，但实际上是不会出现这种情况的，因为如果当前字符是数字，它会被 `while` 循环的第一个 `if` 条件分支捕获。

这个函数现在变得有点复杂了，但它能让我们处理在 Angular 表达式出现的所有数字类型——除了负数，稍后我们会使用 `-` 运算符来处理负数。