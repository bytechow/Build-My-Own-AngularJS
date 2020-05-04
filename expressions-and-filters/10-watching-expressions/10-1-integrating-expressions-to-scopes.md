### 将表达式集成到作用域中去
#### Integrating Expressions to Scopes

以下 `Scope` 对外暴露的方法不仅可以支持原生的函数，同时还支持传入表达式字符串：

- `$watch`
- `$watchCollection`
- `$eval）`（同时还有包括相关的 `$apply` 和 `$evalAsync`）

在内部，`Scope` 会使用 `parse` 函数（也就是之后的 `$parse` 服务）把这些表达式都转换为函数。

由于 `Scope` 依然支持使用原生函数，我们需要检查一下传入的究竟是一个字符串，还是已经是一个函数。我们可以在 `parse` 函数来进行这个检查，如果我们发现要解析的是一个函数，就会把函数原封不动地返回回去：

_test/parse_spec.js_

```js
it('returns the function itself when given one', function() {
  var fn = function() {};
  expect(parse(fn)).toBe(fn);
});
```

在 `parse` 中，我们会根据对参数的检查结果进行不同的处理。如果是字符串的话，我们就对它进行解析。如果是函数，就直接把函数返回回去。如果传入的是其它类型，我们直接返回 `_.noop` 就可以了，`_.noop` 是 LoDash 提供的一个函数，它什么都不会做：

_src/parse.js_

```js
function parse(expr) {
  switch (typeof expr) {
    case 'string':
      // var lexer = new Lexer();
      // var parser = new Parser(lexer);
      // return parser.parse(expr);
    case 'function':
      return expr;
    default:
      return _.noop;
  }
}
```

现在我们要做的第一件事情是在 `scope.js` 中引入 `parse.js` 中的 `parse` 函数：

_src/scope.js_

```js
'use strict';

var _ = require('lodash');
var parse = require('./parse');
```

我们首先要处理是 `$watch` 中的侦听函数（watch function）。我们可以（在 `describe('digest')` 中）创建一个新的测试用例：

_test/scope_spec.js_

```js
it('accepts expressions for watch functions', function() {
  var theValue;

  scope.aValue = 42;
  scope.$watch('aValue', function(newValue, oldValue, scope) {
    theValue = newValue;
  });
  scope.$digest();
  
  expect(theValue).toBe(42);
});
```

要实现这个效果，我们只需要对把传入的侦听函数作为参数调用 `parse` 函数，然后把调用结果替代原来的侦听对象属性就可以了：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  // var self = this;
  // var watcher = {
    watchFn: parse(watchFn),
  //   listenerFn: listenerFn || function() {},
  //   last: initWatchVal,
  //   valueEq: !!valueEq
  // };
  // this.$$watchers.unshift(watcher);
  // this.$$root.$$lastDirtyWatch = null;
  // return function() {
  //   var index = self.$$watchers.indexOf(watcher);
  //   if (index >= 0) {
  //     self.$$watchers.splice(index, 1);
  //     self.$$root.$$lastDirtyWatch = null;
  //   }
  // };
};
```

由于 `$watch` 现在可以传入表达式，`$watchCollection` 自然也是可以的。我们在 `describe('$watchCollection')` 测试板块中加入这个新的测试用例：

_test/scope_spec.js_

```js
it('accepts expressions for watch functions', function() {
  var theValue;

  scope.aColl = [1, 2, 3];
  scope.$watchCollection('aColl', function(newValue, oldValue, scope) {
    theValue = newValue;
  });
  scope.$digest();
  
  expect(theValue).toEqual([1, 2, 3]);
});
```

要让这个用例通过，我们需要调用 `parse` 函数处理 `$watchCollection` 中的侦听函数：

_src/scope.js_

```js
Scope.prototype.$watchCollection = function(watchFn, listenerFn) {
  var self = this;
  var newValue;
  var oldValue;
  var oldLength;
  var veryOldValue;
  var trackVeryOldValue = (listenerFn.length > 1);
  var changeCount = 0;
  var firstRun = true;

  watchFn = parse(watchFn);
  
  // The rest of the function unchanged
};
```

接下来，我们需要处理 `$eval`。在 `describe('$eval')` 板块中加入以下用例：

_test/scope_spec.js_

```js
it('accepts expressions in $eval', function() {
  expect(scope.$eval('42')).toBe(42);
});
```