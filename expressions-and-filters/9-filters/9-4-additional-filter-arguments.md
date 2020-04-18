### 额外的过滤器参数
#### Additional Filter Arguments

我们现在见过的过滤器都只有一个参数：输入表达式的值。但其实过滤器也可以加入额外的参数。这非常有用，因为你可以对单个筛选器进行参数化，使其在不同情况下有不同的行为。

举个例子，如果有一个过滤器，它的作用是把一个字符串重复指定次数，我们就能够利用这个参数指定重复次数。我们在过滤器名字后面加上引号和一个数字就可以了。这个数字会变成过滤器函数的第二个参数：

_test/parse_spec.js_

```js
it('can pass an additional argument to filters', function() {
  register('repeat', function() {
    return function(s, times) {
      return _.repeat(s, times);
    };
  });
  var fn = parse('"hello" | repeat:3');
  expect(fn()).toEqual('hellohellohello');
});
```

你也可以传入多个参数，只需要在每个额外参数的前面都加入冒号就好了：

_test/parse_spec.js_

```js
it('can pass several additional arguments to filters', function() {
  register('surround', function() {
    return function(s, left, right) {
      return left + s + right;
    };
  });
  var fn = parse('"hello" | surround:"*":"!"');
  expect(fn()).toEqual('*hello!');
});
```

现在 Lexer 就可以抛出冒号了（因为我们已经在三元运算符中用到它了）。而 AST 编译器也已经做好处理任意过滤器参数的准备了，因为它本来就是用循环来处理代码的。现在我们需要的就是在 AST 构建处理这些参数。

我们首先要把参数数组 `CallExpression` 提取到一个变量中去：

_src/parse.js_

```js
AST.prototype.filter = function() {
  // var left = this.assignment();
  // while (this.expect('|')) {
    var args = [left];
    // left = {
    //   type: AST.CallExpression,
    //   callee: this.identifier(),
      arguments: args,
  //     filter: true
  //   };
  // }
  // return left;
};
```

现在我们可以使用一个循环来消耗任意（排除过滤器部分的）表达式后面的冒号字符，找到一个就处理一个。每个冒号后面的内容都会作为参数加入到 `args` 数组中。

_src/parse.js_

```js
AST.prototype.filter = function() {
  // var left = this.assignment();
  // while (this.expect('|')) {
  //   var args = [left];
  //   left = {
  //     type: AST.CallExpression,
  //     callee: this.identifier(),
  //     arguments: args,
  //     filter: true
  //   };
    while (this.expect(':')) {
      args.push(this.assignment());
    }
  // }
  // return left;
};
```

这样，解析器就能支持过滤器表达式了！