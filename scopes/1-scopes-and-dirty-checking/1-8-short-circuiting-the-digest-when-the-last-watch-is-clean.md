### 短路优化（Short-Circuiting The Digest When The Last Watch Is Clean）

在目前的代码中，我们会对作用域中 watcher 集合进行持续遍历，直到在某轮遍历中所有 watcher 都未发生变化（或者达到了 TTL 上限）。

鉴于一个 digest 循环可能包含大量的 watcher，尽可能地降低执行 watcher 的次数是非常重要的。这也是为什么我们要对 digest 循环使用一种特有的优化手段。

加入一个作用域上有 100 个 watcher。当我们在作用域上启动 digest 后发现只有第一个 watcher 是“脏”的。也就是说这一个 watch 把这一轮 digest 弄“脏”了，我们不得不再执行一轮 digest。在第二轮中，我们发现没有 watcher 变“脏”后才结束 digest。但这意味着在结束之前，我们需要执行 200 个 watcher！

有一种方法可以让执行量减半，就是对上一个发生变化的 watcher 进行记录。然后，每当我们遇到一个没有发生变化 watcher，我们就看它跟我们记录的 watcher 是否是同一个。如果发现是同一个，说明这一轮已经没有 watcher 变脏了，证明我们没有必要继续执行本轮剩余的 watcher 了，可以马上退出本轮 digest 了。下面的单元测试就是针对这种情况的：

_test/scope_spec.js_

```js
it('ends the digest when the last watch is clean', function() {
  scope.array = _.range(100);
  var watchExecutions = 0;

  _.times(100, function(i) {
    scope.$watch(
      function(scope) {
        watchExecutions++;
        return scope.array[i];
      },
      function(newValue, oldValue, scope) {}
    );
  });
  
  scope.$digest();
  expect(watchExecutions).toBe(200);
  
  scope.array[0] = 420;
  scope.$digest();
  expect(watchExecutions).toBe(301);
});
```

首先，我们在作用域中添加一个有 100 个元素的数组。然后为数组中的每个元素新建一个 watcher。同时我们会加入了一个计数器，每当有 watcher 被执行时就加 1，这样我们就可以记录 watcher 执行的总数了。

然后，我们运行一次 digest 来初始化 watcher。在这一次 digest 中，每个 watcher 会执行两次。

然后，我们对数组的第一个元素进行修改，然后再启动 digest。如果短路优化起作用的话，那么在本次 digest 的第二轮遍历时会发生“短路”，digest 中断，使得最终 watcher 的总执行量为 301 而不是 400。

目前在 `scope_spec.js` 中还没能使用 LoDash，为了接下来能用上它的 `range` 和 `times` 函数，我们需要先引入 LoDash：

_test/scope_spec.js_

```js
'use strict';

var _ = require('lodash');
var Scope = require('../src/scope');
```

正如上面提到的，我们可以通过记录上轮最后一个发生变化的 watcher 来实现短路优化。下面我们对 `Scope` 构造函数添加一个实例属性：

_src/scope.js_

```js
function Scope() {
  this.$$watchers = [];
  this.$$lastDirtyWatch = null;
}
```

现在，无论 digest 在何时启动，我们都会把这个实力属性设为 `null`：

_src/scope.js_

```js
Scope.prototype.$digest = function() {
  // var ttl = 10;
  // var dirty;
  this.$$lastDirtyWatch = null;
  // do {
  //   dirty = this.$$digestOnce();
  //   if (dirty && !(ttl--)) {
  //     throw '10 digest iterations reached';
  //   }
  // } while (dirty);
};
```

在 `$$digestOnce` 方法中，每当我们遇到一个发生了变化的 watcher，我们会把这个 watcher 赋值到这个实例属性上：

_src/scope.js_

```js
Scope.prototype.$$digestOnce = function() {
  // var self = this;
  // var newValue, oldValue, dirty;
  // _.forEach(this.$$watchers, function(watcher) {
  //   newValue = watcher.watchFn(self);
  //   oldValue = watcher.last;
  //   if (newValue !== oldValue) {
      self.$$lastDirtyWatch = watcher;
  //     watcher.last = newValue;
  //     watcher.listenerFn(newValue,
  //       (oldValue === initWatchVal ? newValue : oldValue),
  //       self);
  //     dirty = true;
  //   }
  // });
  // return dirty;
};
```

同时，在 `$$digestOnce` 中，如果我们发现当前的 watcher 跟之前记录在 `$$lastDirtyWatch` 实例属性的 watcher 是同一个的话，我们就会通过返回 `false` 来直接退出当前的循环进程，同时让外层的 `$digest` 循环也停止继续执行：

```js
Scope.prototype.$$digestOnce = function() {
  // var self = this;
  // var newValue, oldValue, dirty;
  // _.forEach(this.$$watchers, function(watcher) {
  //   newValue = watcher.watchFn(self);
  //   oldValue = watcher.last;
  //   if (newValue !== oldValue) {
  //     self.$$lastDirtyWatch = watcher;
  //     watcher.last = newValue;
  //     watcher.listenerFn(newValue,
  //       (oldValue === initWatchVal ? newValue : oldValue),
  //       self);
  //     dirty = true;
    } else if (self.$$lastDirtyWatch === watcher) {
      return false;
    }
  // });
  // return dirty;
};
```

由于我们在这轮遍历中没有发现有变“脏”的 watcher，`dirty` 的值就会是 `undefined`，这也就是 `$$digestOnce` 函数的返回值了。

> 在 `_.forEach` 循环中显式地返回 `false` 会让 LoDash 提前结束并退出循环。

这个优化手段现在就可以起效了。但还有一个极端情况需要处理，就是在 listener 函数注册另一个 watcher:

_test/scope_spec.js_

```js
it('does not end digest so that new watches are not run', function() {
  
  scope.aValue = 'abc';
  scope.counter = 0;
  
  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.$watch(
        function(scope) { return scope.aValue; },
        function(newValue, oldValue, scope) {
          scope.counter++;
        }
      );
    }
  );
  
  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

此时第二个 watcher 并没有执行。原因是在 digest 的第二轮遍历中，在这个新 watcher 运行之前，我们把第一个 watcher 当作是上轮变化了而当前没有再次发生变化的 watcher，因此直接结束了 digest。我们可以通过在添加 watcher 后重置 `$$lastDirtyWatch`来解决这个问题，这样就能有效地禁用短路优化：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn) {
  // var watcher = {
  //   watchFn: watchFn,
  //   listenerFn: listenerFn || function() {},
  //   last: initWatchVal
  // };
  // this.$$watchers.push(watcher);
  this.$$lastDirtyWatch = null;
};
```

现在我们的 digest 周期比之前快得多了。在典型的 Angular 应用中，这个优化并不总是能像上面这个例子一样能如此高效地减少遍历次数。但从平均水平来讲，它产生的效果的已经足以让 Angular 开发团队加入这个优化了。

下面我们再来看看 Angular 实际上是如何对数据进行侦听的。