### 销毁 watcher（Destroying A Watch）

当你注册了一个 watcher 后，你一般会希望只要作用域存在，这个 watcher 就存在，一般不会显式地移除它。但在一些应用场景中，你会希望在保持作用域正在运作的同时，移除某个特定的 watcher。这意味着我们需要添加一个用于移除 watcher 的方法。

Angular 实现移除 watcher 的方法十分聪明：利用了 `$watch` 函数可以返回值的特性。这个值就是一个函数，当调用它的时候会销毁对应的、已经注册了的 watcher。如果用户希望推迟移除 watcher，他们只需要在注册 watcher 之后保存它返回的函数，然后在不再需要 watcher 时候调用这个函数就可以了：

_test/scope_spec.js_

```js
it('allows destroying a $watch with a removal function', function() {
  scope.aValue = 'abc';
  scope.counter = 0;

  var destroyWatch = scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);

  scope.aValue = 'def';
  scope.$digest();
  expect(scope.counter).toBe(2);
  
  scope.aValue = 'ghi';
  destroyWatch();
  scope.$digest();
  expect(scope.counter).toBe(2);
});
```

要实现这个功能，我们需要返回一个函数，这个函数会把当前的 watcher 从 `$$watchers` 中移除出来：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  var self = this;
  // var watcher = {
  //   watchFn: watchFn,
  //   listenerFn: listenerFn,
  //   valueEq: !!valueEq,
  //   last: initWatchVal
  // };
  // self.$$watchers.push(watcher);
  // this.$$lastDirtyWatch = null;
  return function() {
    var index = self.$$watchers.indexOf(watcher);
    if (index >= 0) {
      self.$$watchers.splice(index, 1);
    }
  };
};
```

虽然这样就能移除掉 watcher 本身，但我们还需要处理几个特殊情况，这样才能保证代码实现的健壮性。这些情况都与在 digest 过程中移除 watcher 的场景有关，而这种场景并不少见。

首先，watcher 可能会在自己的 watch 或 listener 函数移除自身。我们要保证它不会影响其他 watcher:

_test/scope.js_

```js
it('allows destroying a $watch during digest', function() {
  scope.aValue = 'abc';

  var watchCalls = [];

  scope.$watch(
    function(scope) {
      watchCalls.push('first');
      return scope.aValue;
    }
  );
  
  var destroyWatch = scope.$watch(
    function(scope) {
      watchCalls.push('second');
      destroyWatch();
    }
  );

  scope.$watch(
    function(scope) {
      watchCalls.push('third');
      return scope.aValue;
    }
  );

  scope.$digest();
  expect(watchCalls).toEqual(['first', 'second', 'third', 'first', 'third']);
});
```

在上面这个单元测试中，我们注册了三个 watcher。第二个的 watcher 会在第一次调用后就移除自身，剩下第一个和第三个 watcher。我们验证 watcher 是否按照正确的顺序进行调用：在 digest 的第一轮中，我们会执行全部三个 watcher 的 watch 函数。然后由于 digest 变“脏”，因此会启动第二轮的执行，但此时第二个 watcher 已经不存在了。

实际上，由于第二个 watcher 在 digest 的第一轮中移除了自己，这时存放 watcher 的数组会自动进行 shift 操作（把第三个 watcher 放到第二个 watcher 的位置），导致 `$$digestOnce` 在那一轮中跳过第三个 watcher。

解决这个问题的诀窍在于对 `$$watchers` 数组进行反向操作，新的 watcher 会被添加到数组的开头，然后按照从后到前的顺序进行遍历。当在 digest 的过程中有 watcher 被移除时，已经执行的 watcher 就会填满空出来的数组空间，这样不会对剩余的 watcher 产生影响。

因此，我们需要在添加 watcher 的时候，使用 `unshift` 代替 `push`：

_src/scope.js_

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  // var self = this;
  // var watcher = {
  //   watchFn: watchFn,
  //   listenerFn: listenerFn || function() {},
  //   last: initWatchVal,
  //   valueEq: !!valueEq
  // };
  this.$$watchers.unshift(watcher);
  // this.$$lastDirtyWatch = null;
  // return function() {
  //   var index = self.$$watchers.indexOf(watcher);
  //   if (index >= 0) {
  //     self.$$watchers.splice(index, 1);
  //   }
  // };
};
```

然后在遍历的时候，我们需要使用 `_.forEachRight` 代替 `_.forEach` 来改变遍历的顺序：

_src/scope.js_

```js
Scope.prototype.$$digestOnce = function() {
  // var self = this;
  // var newValue, oldValue, dirty;
  _.forEachRight(this.$$watchers, function(watcher) {
    // try {
    //   newValue = watcher.watchFn(self);
    //   oldValue = watcher.last;
    //   if (!self.$$areEqual(newValue, oldValue, watcher.valueEq)) {
    //     self.$$lastDirtyWatch = watcher;
    //     watcher.last = (watcher.valueEq ? _.cloneDeep(newValue) : newValue);
    //     watcher.listenerFn(newValue,
    //       (oldValue === initWatchVal ? newValue : oldValue),
    //       self);
    //     dirty = true;
    //   } else if (self.$$lastDirtyWatch === watcher) {
    //     return false;
    //   }
    // } catch (e) {
    //   console.error(e);
    // }
  });
  // return dirty;
};
```