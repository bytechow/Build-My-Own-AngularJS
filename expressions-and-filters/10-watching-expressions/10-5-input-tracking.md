### 对输入表达式进行跟踪
#### Input Tracking

关于表达式侦听，我们可以做一个优化，也就是_输入表达式跟踪_（input tracking）。它的核心概念就是当一个表达式是由一个或多个_输入表达式_组成（比如 `'a * b'` 就是由 `'a'` 和 `'b'`组成），除非其中一个输入表达式发生了变化，否则无需再重新计算表达式。

举个例子，如果有一个数组字面量，它里面所有的元素都没有发生变化：

_test/scope_spec.js_

```js
it('does not re-evaluate an array if its contents do not change', function() {
  var values = [];

  scope.a = 1;
  scope.b = 2;
  scope.c = 3;
  
  scope.$watch('[a, b, c]', function(value) {
    values.push(value);
  });
  
  scope.$digest();
  expect(values.length).toBe(1);
  expect(values[0]).toEqual([1, 2, 3]);
  
  scope.$digest();
  expect(values.length).toBe(1);
  scope.c = 4;
  
  scope.$digest();
  expect(values.length).toBe(2);
  expect(values[1]).toEqual([1, 2, 4]);
});
```

这里我们会对一个所有元素都不是常量的数组进行侦听。我们总共会进行 3 次 digest。第一次 digest 后我们希望 listener 函数返回的是包含了所有元素的数组。第二次 digest 时我们不会对数组的内容进行修改。第三次 digest 之前，我们会对数组内容进行修改，这时我们希望 listener 会被重新触发。

实际上现在程序会抛出一个 “10 $digest iterations reached” 的异常，因为我们使用的基于引用的侦听器，这个侦听器每次执行时都会生成一个新的数组，这是行不通的。

我们需要在 parser 生成的每个表达式函数中记录它的输入表达式——这个表达式改变时可能会让整个表达式的值都发生改变。我们需要对我们的 AST 编译器进行扩展，让它不仅支持处理完整的表达式，还能够处理一个表达式中包含的多个输入表达式。

在介绍 AST 编译器的变动前，我们先来看看侦听器这边应该怎么改。当我们传入一个表达式，如果它既不是常量，也不是单次绑定，而且有输入表达式，它就肯定会有一个 `inputs` 属性。如果是的话，我们将会使用一个特殊的 `inputsWatchDelegate` 委托进行侦听：

_src/parse.js_

```js
function parse(expr) {
  // switch (typeof expr) {
  //   case 'string':
  //     var lexer = new Lexer();
  //     var parser = new Parser(lexer);
  //     var oneTime = false;
  //     if (expr.charAt(0) === ':' && expr.charAt(1) === ':') {
  //       oneTime = true;
  //       expr = expr.substring(2);
  //     }
  //     var parseFn = parser.parse(expr);
  //     if (parseFn.constant) {
  //       parseFn.$$watchDelegate = constantWatchDelegate;
  //     } else if (oneTime) {
  //       parseFn.$$watchDelegate = parseFn.literal ? oneTimeLiteralWatchDelegate :
  //         oneTimeWatchDelegate;
      } else if (parseFn.inputs) {
        parseFn.$$watchDelegate = inputsWatchDelegate;
  //     }
  //     return parseFn;
  //   case 'function':
  //     return expr;
  //   default:
  //     return _.noop;
  // }
}
```

对给定的输入表达式需要单独增加一个侦听器：

_src/parse.js_

```js
function inputsWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  var inputExpressions = watchFn.inputs;

  return scope.$watch(function() {
    
  }, listenerFn, valueEq);
}
```

对输入表达式的跟踪主要是通过维护一个数组，这个数组存放了输入表达式的值。这个数组会使用一些“唯一值”进行初始化（一个空的函数字面量），然后在每次执行 watch 时基于 scope 计算输入表达式的值，从而更新侦听器的值：

_src/parse.js_

```js
function inputsWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  // var inputExpressions = watchFn.inputs;

  var oldValues = _.times(inputExpressions.length, _.constant(function() {}));
  
  // return scope.$watch(function() {
    _.forEach(inputExpressions, function(inputExpr, i) {
      var newValue = inputExpr(scope);
      if (!expressionInputDirtyCheck(newValue, oldValues[i])) {
        oldValues[i] = newValue;
      }
    });
  // }, listenerFn, valueEq);
}
```

我们会把脏值检测的任务委托给一个新的辅助函数，它会对新旧值进行兼容 `NaN` 的、基于引用的相等性判断：

_src/parse.js_

```js
function expressionInputDirtyCheck(newValue, oldValue) {
  return newValue === oldValue ||
    (typeof newValue === 'number' && typeof oldValue === 'number' &&
      isNaN(newValue) && isNaN(oldValue));
}
```

每次执行侦听器函数时，我们都会设置一个 `changed` 的标识，如果任何一个输入表达式发生变化时，这个标识就会被设置为 `true`：

_src/parse.js_

```js
function inputsWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  // var inputExpressions = watchFn.inputs;

  // var oldValues = _.times(inputExpressions.length, _.constant(function() {}));
  
  // return scope.$watch(function() {
    var changed = false;
    // _.forEach(inputExpressions, function(inputExpr, i) {
    //   var newValue = inputExpr(scope);
      if (changed || !expressionInputDirtyCheck(newValue, oldValues[i])) {
        changed = true;
  //       oldValues[i] = newValue;
  //     }
  //   });
  // }, listenerFn, valueEq);
}
```

如果有一个输入表达式发生了改变，组合表达式本身就会生成一个新值，并会把这个新值作为侦听器函数的返回值：

_src/parse.js_

```js
function inputsWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  // var inputExpressions = watchFn.inputs;

  // var oldValues = _.times(inputExpressions.length, _.constant(function() {}));
  var lastResult;
  
  // return scope.$watch(function() {
  //   var changed = false;
  //   _.forEach(inputExpressions, function(inputExpr, i) {
  //     var newValue = inputExpr(scope);
  //     if (changed || !expressionInputDirtyCheck(newValue, oldValues[i])) {
  //       changed = true;
  //       oldValues[i] = newValue;
  //     }
  //   });
    if (changed) {
      lastResult = watchFn(scope);
    }
  //   return lastResult;
  // }, listenerFn, valueEq);
}
```

`lastResult` 变量会保持不变，直到其中至少一个输入表达式发生改变。

> Angular.js 还在 `inputsWatchDelegate`中对只有唯一一个输入表达式的情况做了一个额外的优化。在这种情况下，它会跳过创建 `oldValues` 数组的步骤，以便节省一些内存和计算开销。本书会跳过这个优化。

在处理了侦听委托之后，让我们考虑一下它使用的 `inputs` 数组是如何实现的。我们需要把目光转回 AST 编译器。

要组成 `inputs`，首先需要判断输入表达式到底是什么类型。不同类型的表达式会有不同的输入类型，因此需要分别确定各个 AST 节点类型的输入节点。这意味着我们需要有一个树遍历函数，跟之前我们检查表达式是否为常量时创建的那个函数类似。

实际上，我们不需要再新建一个树遍历函数，只需要对现有的这个函数进行扩展，让它既可以检查常量，又可以对输入表达式进行校验就可以了。首先我们需要对它进行重命名，让它的名字与行为之间更为贴切：

_src/parse.js_

```js
function markConstantAndWatchExpressions(ast) {
  // var allConstants;
  // switch (ast.type) {
    case AST.Program:
  //     allConstants = true;
  //     _.forEach(ast.body, function(expr) {
        markConstantAndWatchExpressions(expr);
    //     allConstants = allConstants && expr.constant;
    //   });
    //   ast.constant = allConstants;
    //   break;
    // case AST.Literal:
    //   ast.constant = true;
    //   break;
    // case AST.Identifier:
    //   ast.constant = false;
    //   break;
    case AST.ArrayExpression:
      // allConstants = true;
      // _.forEach(ast.elements, function(element) {
        markConstantAndWatchExpressions(element);
    //     allConstants = allConstants && element.constant;
    //   });
    //   ast.constant = allConstants;
    //   break;
    case AST.ObjectExpression:
    //   allConstants = true;
    //   _.forEach(ast.properties, function(property) {
        markConstantAndWatchExpressions(property.value);
    //     allConstants = allConstants && property.value.constant;
    //   });
    //   ast.constant = allConstants;
    //   break;
    // case AST.ThisExpression:
    // case AST.LocalsExpression:
    //   ast.constant = false;
    //   break;
    case AST.MemberExpression:
      markConstantAndWatchExpressions(ast.object);
      // if (ast.computed) {
        markConstantAndWatchExpressions(ast.property);
      // }
      // ast.constant = ast.object.constant &&
      //   (!ast.computed || ast.property.constant);
      // break;
    case AST.CallExpression:
      // allConstants = ast.filter ? true : false;
      // _.forEach(ast.arguments, function(arg) {
        markConstantAndWatchExpressions(arg);
      //   allConstants = allConstants && arg.constant;
      // });
      // ast.constant = allConstants;
      // break;
    case AST.AssignmentExpression:
      markConstantAndWatchExpressions(ast.left);
      markConstantAndWatchExpressions(ast.right);
      // ast.constant = ast.left.constant && ast.right.constant;
      // break;
    case AST.UnaryExpression:
      markConstantAndWatchExpressions(ast.argument);
      // ast.constant = ast.argument.constant;
      // break;
    case AST.BinaryExpression:
    case AST.LogicalExpression:
      markConstantAndWatchExpressions(ast.left);
      markConstantAndWatchExpressions(ast.right);
      // ast.constant = ast.left.constant && ast.right.constant;
      // break;
    case AST.ConditionalExpression:
      markConstantAndWatchExpressions(ast.test);
      markConstantAndWatchExpressions(ast.consequent);
      markConstantAndWatchExpressions(ast.alternate);
  //     ast.constant =
  //       ast.test.constant && ast.consequent.constant && ast.alternate.constant;
  //     break;
  // }
}
```

在 `ASTCompiler.compile` 方法中也需要更改对应的函数名称：

_src/parse.js_

```js
ASTCompiler.prototype.compile = function(text) {
  // var ast = this.astBuilder.ast(text);
  markConstantAndWatchExpressions(ast);
  // ...
};
```

现在这个函数除了标记常量，我们还需要将每个 AST 节点的输入节点收集到一个叫 `toWatch` 的属性中。我们先要定义一个一个变量来收集输入节点：

_src/parse.js_

```js
function markConstantAndWatchExpressions(ast) {
  // var allConstants;
  var argsToWatch;
  // ...
}
```

现在，我们需要依次考虑每个节点类型的输入，想想“这个表达式的值会在什么时候发生变化？”

简单的字面量并不需要被侦听——它们永远都不会被改变：

_src/parse.js_

```js
case AST.Literal:
  // ast.constant = true;
  ast.toWatch = [];
  // break;
```

对标识符表达式来说，我们需要侦听的就是表达式本身。它没有更小的部门可以被拆解了：

_src/parse.js_

```js
case AST.Identifier:
  // ast.constant = false;
  ast.toWatch = [ast];
  // break;
```

对数组来说，我们需要对所有非常量元素对应的输入表达式进行侦听：

_src/parse.js_

```js
case AST.ArrayExpression:
  // allConstants = true;
  argsToWatch = [];
  // _.forEach(ast.elements, function(element) {
  //   markConstantAndWatchExpressions(element);
  //   allConstants = allConstants && element.constant;
    if (!element.constant) {
      argsToWatch.push.apply(argsToWatch, element.toWatch);
    }
  // });
  // ast.constant = allConstants;
  ast.toWatch = argsToWatch;
  // break;
```

同样地，对象会基于非常量属性值进行侦听：

_src/parse.js_

```js
case AST.ObjectExpression:
  // allConstants = true;
  argsToWatch = [];
  // _.forEach(ast.properties, function(property) {
  //   markConstantAndWatchExpressions(property.value);
  //   allConstants = allConstants && property.value.constant;
    if (!property.value.constant) {
      argsToWatch.push.apply(argsToWatch, property.value.toWatch);
    }
  // });
  // ast.constant = allConstants;
  ast.toWatch = argsToWatch;
  // break;
```