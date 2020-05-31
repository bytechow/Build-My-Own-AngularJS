### 有状态的过滤器
#### Stateful Filters

由于我们本章已经实现了常量优化和输入项跟踪，我们已经看到了过滤器函数调用是如何与普通函数区分开来的：如果它们的参数是常量，过滤器表达式也会是常量，只会监听它们输入项的变化。

关于这个实现，我们有一个相当重要的假设，就是如果一个过滤器的输入项不变，那它的结果值就不变。换句话说，我们认为过滤器是一个[纯函数](https://en.wikipedia.org/wiki/Pure_function)（pure functions）。

这个规则对于大多数过滤器来说（包括我们之前的 filter 过滤器）都是适用的。不管你调用的是不是 filter 过滤器、或者调用多少次，只要传入的值相同，那返回的值就是一样的。对于函数来说，这是一个很好的属性，因为它使函数更容易理解，也因为它允许我们进行各种优化：当在表达式中使用 filter 过滤器时，除非输入项发生了改变（或者其他参数中的一个参数发生改变），否则我们不需要对这个表达式进行再次计算。这对于很多应用来说都是一个显著的优化。

然而，有时这个假设并不成立。我们可以想象一下，有一个过滤器，它的值会改变即使它的输入项没有改变。这种过滤器的一个例子是在表达式输出中嵌入当前时间的过滤器。Angular 允许你在这种类型的过滤器中加入一个特殊的 `$stateful` 属性。如果你把这个属性设置为 `true`，常量和输入项优化就不会应用到这个过滤器上：

_test/scope_spec.js_

```js
it('allows $stateful filter value to change over time', function(done) {
  
  register('withTime', function() {
    return _.extend(function(v) {
      return new Date().toISOString() + ': ' + v;
    }, {
      $stateful: true
    });
  });

  var listenerSpy = jasmine.createSpy();
  scope.$watch('42 | withTime', listenerSpy);
  scope.$digest();
  var firstValue = listenerSpy.calls.mostRecent().args[0];
  
  setTimeout(function() {
    scope.$digest();
    var secondValue = listenerSpy.calls.mostRecent().args[0];
    expect(secondValue).not.toEqual(firstValue);
    done();
  }, 100);
});
```

由于在这个测试用例中用到 `register` 函数，所以我们需要在 `scope_spec.js` 文件中引入这个函数：

_test/scope_spec.js_

```js
'use strict';
var _ = require('lodash');
var Scope = require('../src/scope');
var register = require('../src/filter').register;
```

这个 watch 每次运行都会计算出相同的值，测试未能通过。这是因为它包含了一个常量输入项，它并不会发生改变。我们需要在 `parse.js` 中做一些改变，对于有状态的过滤器调用，我们会禁用掉常量和输入项优化。

_src/parse.js_

```js
case AST.CallExpression:
  var stateless = ast.filter && !filter(ast.callee.name).$stateful;
  allConstants = stateless ? true : false;
  argsToWatch = [];
  _.forEach(ast.arguments, function(arg) {
    markConstantAndWatchExpressions(arg);
    allConstants = allConstants && arg.constant;
    if (!arg.constant) {
      argsToWatch.push.apply(argsToWatch, arg.toWatch);
    }
  });
  ast.constant = allConstants;
  ast.toWatch = stateless ? argsToWatch : [ast];
  break;
```