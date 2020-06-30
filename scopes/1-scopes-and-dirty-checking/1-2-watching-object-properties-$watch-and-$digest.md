### 侦听对象属性的方法：$watch 和 $digest（Watching Object Properties: $watch And $digest）

`$watch` 和 `$digest`是一个硬币的两面。它们共同组成了 digest 循环的核心：对数据变化作出反应。

有了 `$watch` 你可以把一个叫 _watcher_ 的东西添加到作用域上。当作用域发生了变化时，watcher 会得到通知。我们可以通过调用 `$watch` 时传入两个参数来创建一个 watcher，而这两个参数都必须为函数：

- 一个是 watch 函数，它指定了我们要侦听的数据。
- 另一个是 listener 函数，当要侦听的数据发生变化时，就会调用这个函数。

> 实际上，Angular 使用者一般会用一个 watch 表达式指定要侦听的数据，而不是用 watch 函数。watch 表达式实际上就是一个字符串，比如“user.firstName”，这个数据来源于数据绑定，也可能来源于一个指令属性，或者就是在 JavaScript 代码中定义的。Angular 会在内部对它进行解析和编译，最终同样是转化成一个 watch 函数。我们会在本书第二部分实现这一功能，在此之前，我们先使用 watch 函数这种较低级别的方法进行指定。

硬币的另一面就是 `$digest` 函数了。这个函数会对所有绑定到作用域上的 watcher 进行遍历，对各个 watcher 上的 watch 函数和对应的 listener 函数进行调用。

要实现这两个部件，我们先要定义一个测试用例，这个用例会嘉定我们可以使用 `$watch` 函数来注册 watcher，并且当我们调用 `$digest` 时，这个 watcher 的 listener 函数会被调用。

为了方便管理，我们在 `scope_spec.js` 文件中加入一个嵌套 `describe` 代码块：

_test/scope_spec.js_

```js
describe('Scope', function() {

  it('can be constructed and used as an object', function() {
    var scope = new Scope();
    scope.aProperty = 1;
    
    expect(scope.aProperty).toBe(1);
  });

  describe('digest', function() {
  
    var scope;
  
    beforeEach(function() {
      scope = new Scope();
    });
  
    it('calls the listener function of a watch on first $digest', function() {
      var watchFn = function() { return 'wat'; };
      var listenerFn = jasmine.createSpy();
      scope.$watch(watchFn, listenerFn);
  
      scope.$digest();
  
      expect(listenerFn).toHaveBeenCalled();
    });

  });
  
});
```

在上面的测试用例中，我们调用 `$watch` 为作用域注册了一个 watcher。由于目前我们的关注点不在 watch 函数参数上，传入一个会返回常量的函数就可以了。而对于 listener 函数参数，我们会传入一个 [Jasmine Spy](https://jasmine.github.io/2.0/introduction.html#section-Spies)。然后我们只需要调用 `$digest` 并检查一下 listener 函数有没有被调用就可以了。

> spy 是一个 Jasmine 术语，表示一种模拟函数。这种技术可以让我们方便地验证诸如“这个函数会被调用吗？”或“调用这个函数时传入的参数是什么？”的问题。

要让这个测试用例通过，我们需要做几件事。首先，作用域需要有一个能保存所有 watcher 的空间。我们会在 `Scope` 构造函数中添加一个数组来存放它们：

_src/scope.js_

```js
function Scope() {
  this.$$watchers = [];
}
```

> 两个美元符号组成的前缀——`$$` ，表示这个变量仅用于 Angular 框架内部，不允许在应用代码中进行访问。

现在我们可以定义 `$watch` 函数了。这个函数会接收两个函数类型的参数，并把这两个参数存放到 `$$watchers` 数组中。我们希望每个作用域对象都有 `$watch` 函数，因此会把它添加到 `Scope` 构造函数的原型上：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
  var watcher = {
    watchFn: watchFn,
    listenerFn: listenerFn
  };
  this.$$watchers.push(watcher);
};
```

最后我再来实现 `$digest` 函数。现在我们要实现的是 `$digest` 函数的简单版本，它只需要遍历所有已注册的 watcher 并调用它们的 listener 函数就可以了：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  _.forEach(this.$$watchers, function(watcher) {
    watcher.listenerFn();
  });
};
```

这个函数里面用到的 `forEach` 函数来源于 LoDash，所以，我们需要在文件的顶部引入 LoDash：

_src/scope.js_

```js
'use strict';

var _ = require('lodash');

// ...
```

单元测试是通过了，但目前 `$digest` 的用处不大。我们真正的需求是，检查 watch 函数指定的数据是否发生了改变，如果发生了改变，我们才调用它的 listener 函数。这才可以称之为 _脏值检测_（dirty-checking）。