### 方法调用
#### Method Calls

在 JavaScript 中，可以调用的并不总是一个单纯的函数。当把函数作为属性添加到对象上，并使用点运算符或方括号把函数从对象中引用出来调用时，这个函数就是作为一个_方法_（method）被调用的。这意味着在函数体中使用 `this` 关键词会指向包含这个方法的对象。因此，在下面的单元测试中，由于在表达式中采用了上述调用方法，`aFunction` 中的 `this` 关键词应该要指向 `aMember`。先来看看使用计算属性的方式进行调用的情况：

_test/parse_spec.js_

```js
it('calls methods accessed as computed properties', function() {
  var scope = {
    anObject: {
      aMember: 42,
      aFunction: function() {
        return this.aMember;
      }
    }
  };
  var fn = parse('anObject["aFunction"]()');
  expect(fn(scope)).toBe(42);
});
```

然后是使用非计算属性的方式进行调用：

_test/parse_spec.js_

```js
it('calls methods accessed as non-computed properties', function() {
  var scope = {
    anObject: {
      aMember: 42,
      aFunction: function() {
        return this.aMember;
      }
    }
  };
  var fn = parse('anObject.aFunction()');
  expect(fn(scope)).toBe(42);
});
```

完成这项工作的所有步骤都在 AST 编译器中。我们需要在递归函数 `recurse` 中的 `CallExpression` 中生成正确的 JavaScript 代码，这样 `this` 才能绑定到原始 Angular 表达式中引用方法的对象上去。

实现的关键在于引入一个“调用上下文”对象，这个对象会保存有关方法调用的信息。我们会在调用函数的表达式中加入这个对象。当我们处理被调用函数时，会把这个对象作为 `recurse` 方法的第二个参数传入。然后 `recurse` 会把我们需要的信息添加到这个对象中：

_src/parse.js_

```js
case AST.CallExpression:
  var callContext = {};
  var callee = this.recurse(ast.callee, callContext);
  // var args = _.map(ast.arguments, _.bind(function(arg) {
  //   return this.recurse(arg);
  // }, this));
  // return callee + '&&' + callee + '(' + args.join(',') + ')';
```

在声明 `recurse` 方法是，我们需要让它能接收第二个参数，参数名称为 `context`：

_src/parse.js_

```js
ASTCompiler.prototype.recurse = function(ast, context) {
  // ...
};
````

当我们会把 context 参数传递给 `recurse` 时，我们希望发生的是，如果现在处理的是一个方法调用，我们会在 context 上填充三个属性：

- `context` - 拥有方法的对象。最终会成为函数体中的 `this`。
- `name` - 该方法在对象中的属性名称。
- `computed` - 这个方法是否用计算属性的方式进行访问。

`CallExpression` 分支会利用这三个属性来构造被调用函数，以使得最终生成的调用表达式使用到正确的 `this`：

_src/parse.js_

```js
case AST.CallExpression:
  // var callContext = {};
  // var callee = this.recurse(ast.callee, callContext);
  // var args = _.map(ast.arguments, _.bind(function(arg) {
  //   return this.recurse(arg);
  // }, this));
  if (callContext.name) {
    if (callContext.computed) {
      callee = this.computedMember(callContext.context, callContext.name);
    } else {
      callee = this.nonComputedMember(callContext.context, callContext.name);
    }
  }
  // return callee + '&&' + callee + '(' + args.join(',') + ')';
```

这里发生的事情是，我们会在 JavaScript 代码中重新构造计算或非计算属性的访问，以便绑定 `this`。

现在，我们已经看到如何使用调用上下文（call context）了，我们需要在 `MemberExpression` 分支中对调用上下文进行赋值了，这里是构建方法调用表达式的被调用函数的地方。context 参数的 `context` 属性应该是成员表达式的所属对象：

_src/parse.js_

```js
case AST.MemberExpression:
  intoId = this.nextId();
  var left = this.recurse(ast.object);
  if (context) {
    context.context = left;
  }
  if (ast.computed) {
    var right = this.recurse(ast.property);
    this.if_(left,
      this.assign(intoId, this.computedMember(left, right)));
  } else {
    this.if_(left,
      this.assign(intoId, this.nonComputedMember(left, ast.property.name)));
  }
  return intoId;
```

根据查找对象属性的方式不同（是计算的还是非计算的），context 参数的 `name` 和 `computed` 属性也不同：

_src/parse.js_

```js
case AST.MemberExpression:
  // intoId = this.nextId();
  // var left = this.recurse(ast.object);
  // if (context) {
  //   context.context = left;
  // }
  // if (ast.computed) {
  //   var right = this.recurse(ast.property);
  //   this.if_(left,
  //     this.assign(intoId, this.computedMember(left, right)));
    if (context) {
      context.name = right;
      context.computed = true;
    }
  // } else {
  //   this.if_(left,
  //     this.assign(intoId, this.nonComputedMember(left, ast.property.name)));
    if (context) {
      context.name = ast.property.name;
      context.computed = false;
    }
  // }
  // return intoId;
```

现在我们就能正确地生成调用方法的代码，然后测试套件就能通过了。

与方法调用密切相关的一个特性与非方法函数的 `this` 有关。当你在 Angular 表达式中调用一个单独的函数时，它的 `this` 实际上会被绑定到 Scope 对象上：

```js
it('binds bare functions to the scope', function() {
  var scope = {
    aFunction: function() {
      return this;
    }
  };
  var fn = parse('aFunction()');
  expect(fn(scope)).toBe(scope);
});
```

这里有一个例外情况，当函数是放到表达式的本地变量而非 Scope 时，`this` 会指向本地变量：

_test/parse_spec.js_

```js
it('binds bare functions on locals to the locals', function() {
  var scope = {};
  var locals = {
    aFunction: function() {
      return this;
    }
  };
  var fn = parse('aFunction()');
  expect(fn(scope, locals)).toBe(locals);
});
```

我们可以通过在 `Identifier` 表达式中对上下文也进行赋值来实现。这里的上下文不是 `l` 就是 `s`，名称是标识符的名称，而 `computed` 属性总是 `false`：

_src/parse.js_

```js
case AST.Identifier:
  // intoId = this.nextId();
  // this.if_(this.getHasOwnProperty('l', ast.name),
  //   this.assign(intoId, this.nonComputedMember('l', ast.name)));
  // this.if_(this.not(this.getHasOwnProperty('l', ast.name)) + ' && s',
  //   this.assign(intoId, this.nonComputedMember('s', ast.name)));
  if (context) {
    context.context = this.getHasOwnProperty('l', ast.name) + '?l:s';
    context.name = ast.name;
    context.computed = false;
  }
  // return intoId;
```