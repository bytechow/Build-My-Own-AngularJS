### 单次绑定表达式
#### One-Time Expressions

作为 Angular 开发者，我们经常会遇到有些 watch 只需要赋值一次，后续不会再发生改变的情况。比较常见的就是像下面这样的对象列表：

```xml
<li ng-repeat="user in users">
  {{user.firstName}} {{user.lastName}}
</li>
```

这段代码使用的 `ng-repeat` 指定首先会为 `users` 生成了一个侦听器集合，但对于每个 `user` 来说，它同时也会增加两个 watcher——第一个是姓（first name），第二个是名（last name）。

一般来说，这样的姓名都不会发生改变，因为它们本身就是只读的——没有任何应用逻辑会去修改它。然而，Angular 现在还是必须在每次 digest 时对它们进行脏值检测，因为 digest 并不知道你不需要对它们进行改变。这些无意义的脏值检测可能会在大型应用和低端设备上造成明显的性能影响。

为了解决这个问题，Angular 提供了单次绑定（one-time binding）的特性。当你知道一个监听器在存续期间的值不会改变时，你可以在监听表达式前面加入两个冒号，以便让 Angular 知道：

```xml
<li ng-repeat="user in users">
  {{::user.firstName}} {{::user.lastName}}
</li>
```

单次绑定监听语法是在表达式引擎中实现的，因为我们可以在任何监听表达式中使用：

_test/scope_spec.js_

```js
it('accepts one-time watches', function() {
  var theValue;

  scope.aValue = 42;
  scope.$watch('::aValue', function(newValue, oldValue, scope) {
    theValue = newValue;
  });
  scope.$digest();
  
  expect(theValue).toBe(42);
});
```

单次绑定的监听器与普通的监听器最大的区别在于，当 watcher 完成运算后会自动销毁，这样它就不会再对 digest 循环造成压力了：

_test/scope_spec.js_

```js
it('removes one-time watches after first invocation', function() {
  scope.aValue = 42;
  scope.$watch('::aValue', function() {});
  scope.$digest();
  
  expect(scope.$$watchers.length).toBe(0);
});
```

单次绑定的监听的实现都在 `parse.js` 里，利用上一个节中介绍的监听委托系统。如果表达式前面有两个冒号，我们就把“单次绑定的监听委托”（one-time watch delegate）作为监听委托：

_src/parse.js_

```js
function parse(expr) {
  // switch (typeof expr) {
  //   case 'string':
  //     var lexer = new Lexer();
  //     var parser = new Parser(lexer);
      var oneTime = false;
      if (expr.charAt(0) === ':' && expr.charAt(1) === ':') {
        oneTime = true;
        expr = expr.substring(2);
      }
      // var parseFn = parser.parse(expr);
      // if (parseFn.constant) {
      //   parseFn.$$watchDelegate = constantWatchDelegate;
      } else if (oneTime) {
        parseFn.$$watchDelegate = oneTimeWatchDelegate;
  //     }
  //     return parseFn;
  //   case 'function':
  //     return expr;
  //   default:
  //     return _.noop;
  // }
}
```

从表面上看，单次绑定的行为跟常量侦听委托一模一样：运行一次监听器，然后移除它。确定，如果我们按照常量侦听委托的方法来实现单次绑定监听委托缺失可以通过测试用例：

_src/parse.js_

```js
function oneTimeWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  var unwatch = scope.$watch(function() {
    return watchFn(scope);
  }, function(newValue, oldValue, scope) {
    if (_.isFunction(listenerFn)) {
      listenerFn.apply(this, arguments);
    }
    unwatch();
  }, valueEq);
  return unwatch;
}
```

但这里还有一个问题，单次绑定表达式在第一次运算时可能并不会像常量表达式那样总是存在着值。举个例子，我们在第一次表达式运算时可能还在等待服务器返回数据。如果单次绑定能够支持这种异步的场景，会更有用。这也是为什么它们只会在返回值不为 `undefined` 时被移除：

_test/scope_spec.js_

```js
it('does not remove one-time-watches until value is defined', function() {
  scope.$watch('::aValue', function() {});
  
  scope.$digest();
  expect(scope.$$watchers.length).toBe(1);
  
  scope.aValue = 42;
  scope.$digest();
  expect(scope.$$watchers.length).toBe(0);
});
```

要通过这个测试用例，我们可以在调用 `unwatch` 语句的外面包裹一层 `if` 语句：

_src/parse.js_

```js
function oneTimeWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  // var unwatch = scope.$watch(
  //   function() {
  //     return watchFn(scope);
  //   },
  //   function(newValue, oldValue, scope) {
  //     if (_.isFunction(listenerFn)) {
  //       listenerFn.apply(this, arguments);
  //     }
      if (!_.isUndefined(newValue)) {
        // unwatch();
      }
  //   }, valueEq);
  // return unwatch;
}
```

但这还并不是最优的处理方式。我们之前也看到了，在 digest 时可能会发生很多事情，有可能其中一件事情就是把表达式的值再次变成 `undefined`。Angular 只会在值_稳定_下来后才会移除单次绑定表达式的 watch，换言之，这个值在 digest 的最后时刻一定不能是 `undefined`（才会去移除）：

_test/scope_spec.js_

```js
it('does not remove one-time-watches until value stays defined', function() {
  scope.aValue = 42;

  scope.$watch('::aValue', function() {});
  var unwatchDeleter = scope.$watch('aValue', function() {
    delete scope.aValue;
  });
  
  scope.$digest();
  expect(scope.$$watchers.length).toBe(2);
  
  scope.aValue = 42;
  unwatchDeleter();
  scope.$digest();
  expect(scope.$$watchers.length).toBe(0);
});
```

在这个测试用例中，我们会加入第二个侦听器，它会在 digest 期间把单词绑定的监听器的值在此变成 `undefined`。只要第二个监听器存在，单次绑定的侦听器就不会稳定下来，也就不会被移除了。

我们必须要对单次绑定的监听器的最后一个值进行跟踪，然后在 digest 结束时看看它是否已经被定义过了。如果已经定义过，我们才会移除这个侦听器。我们可以利用 `$$postDigest` 方法来延迟执行移除的操作：

_src/parse.js_

```js
function oneTimeWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  var lastValue;
  // var unwatch = scope.$watch(function() {
  //   return watchFn(scope);
  // }, function(newValue, oldValue, scope) {
    lastValue = newValue;
    // if (_.isFunction(listenerFn)) {
    //   listenerFn.apply(this, arguments);
    // }
    // if (!_.isUndefined(newValue)) {
      scope.$$postDigest(function() {
        if (!_.isUndefined(lastValue)) {
          // unwatch();
        }
      });
  //   }
  // }, valueEq);
  // return unwatch;
}
```

现在效果已经很不错了，但我们还需要处理单次绑定侦听器的一种特殊情况：在处理表示集合的字面量（比如数组或者对象）时，单次绑定在移除这个侦听器之前，需要检查字面量里面的值是否都被定义过了。举个例子，如果要单次绑定的是一个数组字面量，我们只会在所有数组元素都不为 `undefined` 时才移除它：

_test/scope_spec.js_

```js
it('does not remove one-time watches before all array items defined', function() {
  scope.$watch('::[1, 2, aValue]', function() {}, true);

  scope.$digest();
  expect(scope.$$watchers.length).toBe(1);
  
  scope.aValue = 3;
  scope.$digest();
  expect(scope.$$watchers.length).toBe(0);
});
```

对象字面量的侦听器也一样，只有在对象里所有属性值都不为 `undefined` 时才移除这个侦听器：

_test/scope_spec.js_

```js
it('does not remove one-time watches before all object vals defined', function() {
  scope.$watch('::{a: 1, b: aValue}', function() {}, true);
  
  scope.$digest();
  expect(scope.$$watchers.length).toBe(1);
  
  scope.aValue = 3;
  scope.$digest();
  expect(scope.$$watchers.length).toBe(0);
});
```

> 这个特性也支持指令使用对象字面量进行单次绑定，比如 `ngClass` 和 `ngClass`。

如果表达式是一个字面亮，我们应该使用一个特殊的“一次性字面量侦听器”委托来代替普通的单次绑定侦听委托：

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
        parseFn.$$watchDelegate = parseFn.literal ? oneTimeLiteralWatchDelegate : oneTimeWatchDelegate;
  //     }
  //     return parseFn;
  //   case 'function':
  //     return expr;
  //   default:
  //     return _.noop;
  // }
}
```

这个委托函数跟单词绑定的侦听委托相似，但它并没有检查表达式本身的值是否已经被定义，同时会假设这个字面亮是一个集合，最后检查这个集合中包含的元素是否都已被定义：

_src/parse.js_

```js
function oneTimeLiteralWatchDelegate(scope, listenerFn, valueEq, watchFn) {
  function isAllDefined(val) {
    return !_.any(val, _.isUndefined);
  }
  var unwatch = scope.$watch(function() {
    return watchFn(scope);
  }, function(newValue, oldValue, scope) {
    if (_.isFunction(listenerFn)) {
      listenerFn.apply(this, arguments);
    }
    if (isAllDefined(newValue)) {
      scope.$$postDigest(function() {
        if (isAllDefined(newValue)) {
          unwatch();
        }
      });
    }
  }, valueEq);
  return unwatch;
}
```

> 数字、字符串、布尔值和 `undefined` 也都是字面量，但它们永远都不会被传递给 `oneTimeLiteralWatchDelegate`，因为它们都是常量，会直接被委托给 `constantWatchDelegate`。换句话说，一次性字面量委托函数（one-time literal delegate）只会接收到至少包含一个非常量元素的数组或者对象。