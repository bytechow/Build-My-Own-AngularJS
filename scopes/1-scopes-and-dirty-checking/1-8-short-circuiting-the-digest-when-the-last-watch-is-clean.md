### 短路优化（Short-Circuiting The Digest When The Last Watch Is Clean）

在目前的代码中，我们会对作用域中的所有 watcher 进行多轮遍历，直到在某轮遍历中所有 watcher 都未发生变化（或者达到了 TTL 上限）。

由于一个 digest 可能需要执行包含大量的 watcher，尽可能地降低执行 watcher 的次数变得尤为重要。这也是为什么我们要对 digest 循环应用一种特殊的优化措施——短路优化了。

假如一个作用域上有 100 个 watcher，当在作用域上启动 digest 后发现只有第一个 watcher 的值发生了变化，也就是说这一个 watch （老鼠屎）把这一轮 digest（粥）弄“脏”了，我们不得不再执行一轮 digest，在第二轮遍历中，我们发现所有 watcher 的值都不再发生变化了，这时才能结束 digest。但这个过程我们需要执行 200 个 watcher！

其实有一种方法可以让执行量减半，那就是对上一个发生变化的 watcher  进行记录。然后，每当我们遇到一个没有发生变化 watcher，我们就看它跟上一轮中记录的最后一个发生变化的 watcher 是否是同一个。如果发现是同一个，说明这一轮已经没有 watcher 会发生变化了，我们也就没有必要继续执行本轮剩余的 watcher ，可以马上结束 digest 了。下面这个单元测试就是针对这种情况建立的：

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

首先，我们在作用域中添加一个有 100 个元素的数组属性，然后为数组中的每个元素创建一个 watcher，同时还会加入了一个计数器，每当 watcher 被执行时，计数器就会加 1，这样我们就可以记录 watcher 执行的总数了。

然后，我们先运行一次 digest 来初始化 watcher。在这一次 digest 中，每个 watcher 会执行两次。

然后，我们对数组的第一个元素进行修改，然后再启动 digest。如果短路优化起作用的话，那么这次 digest 的第二轮遍历到第一个元素时，程序会发生“短路”，中断 digest，使得最终 watcher 的总执行次数为 301 而不是 400。

`scope_spec.js` 现在还未引入 LoDash，为了使用它的 `range` 和 `times` 函数，我们需要先引入 LoDash：

_test/scope_spec.js_

```js
'use strict';

var _ = require('lodash');
var Scope = require('../src/scope');
```

正如上面提到的，我们可以通过记录上轮最后一个发生变化的 watcher 来实现这种优化。我们先在 `Scope` 构造函数中添加一个实例属性 `$$lastDirtyWatch`：

_src/scope.js_

```js
function Scope() {
  this.$$watchers = [];
  this.$$lastDirtyWatch = null;
}
```

只要 digest 启动，我们就会把这个实例属性重置为 `null`：

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

在 `$$digestOnce` 方法中，每当我们遇到一个发生了变化的 watcher，我们会把这个 watcher 赋值给这个实例属性：

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

同时，在 `$$digestOnce` 中，如果我们发现当前的 watcher 跟之前记录在 `$$lastDirtyWatch` 实例属性的 watcher 是同一个的话，我们就会通过 `return false` 的方式来直接退出当前的遍历轮次，同时也会退出外层的 `$digest` 循环：

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

由于我们在这轮遍历中不会再遇到变“脏”了的 watcher，`dirty` 的值就会是 `undefined`，这同时也是 `$$digestOnce` 函数的返回值（所以才会同时退出外层的 digest 循环）。

> 在 `_.forEach` 循环中显式地返回 `false` 会让 LoDash 提前结束并退出循环。

这个优化手段现在就可以起效了。但我们还需要处理一种极端情况——在 listener 函数中注册另一个 watcher:

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

这里我们注册的第二个 watcher 并没有被执行。原因是在 digest 的第二轮遍历中，在这个新增的 watch 函数运行之前，程序把第一个 watcher 当作是上轮变化了而本轮没有再次发生变化的 watcher（符合短路优化条件），短路优化直接结束了 digest。我们可以通过在注册 watcher 后重置 `$$lastDirtyWatch`来解决这个问题，这样就能显式地禁用短路优化：

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

现在我们的 digest 循环就比之前快得多了。对于一般的 Angular 应用来说，这个优化并不总是能像上面这个例子一样能高效地减少遍历次数。但从平均水平上来看，它产生的效果已经足以让 Angular 开发团队加入这个优化了。

下面的章节，我们将回归到 Angular 实际上是如何对数据进行侦听的话题上来。