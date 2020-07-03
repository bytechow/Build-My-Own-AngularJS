### 销毁 watcher（Destroying A Watch）

在注册 watcher 以后，我们希望只要作用域存在，这个 watcher 就存在，一般不会主动移除它。但有时我们也希望在保证作用域正常运作的同时，移除某个 watcher。这意味着我们需要添加一个用于移除 watcher 的方法。

Angular 实现移除 watcher 的方式十分聪明：Angular 中的 `$watch` 函数会有一个返回值。这个值就是一个函数，当它被调用的时候就会销毁对应的 watcher。如果用户希望推迟移除 watcher 的时机，他们只需要在注册 watcher 之后保存它返回的函数 ，然后在不再需要 watcher 的时候调用这个函数就可以了：

_test/scope\_spec.js_

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

如上所述，要实现这个功能，我们需要返回一个函数，这个函数会在 `$$watchers` 中移除这个侦听器：

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

虽然这样确实能移除 watcher ，但我们还需要处理几个特殊情况以保证代码的健壮性。这几种特殊情况都与在 digest 过程中移除 watcher 的这种操作有关，这种场景并不少见。

首先，watcher 可能会在自己的 watch 或 listener 函数移除自身。我们要保证它不会影响其他 watcher:

_test/scope\_spec.js_

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

在上面这个单元测试中，我们注册了三个 watcher。其中第二个 watcher 会在第一次调用之后就移除了自身，剩下第一个和第三个 watcher。我们会验证 watcher 是否按照正确的顺序进行调用：在 digest 的第一轮中，我们希望三个 watcher 的 watch 函数都会被执行。由于 digest 变“脏”，会执行第二轮的遍历，但此时第二个 watcher 已经不存在了。

实际上，由于第二个 watcher 在 digest 的第一轮中移除了自己，这时存放 watcher 的数组会自动进行 shift 操作（把第三个 watcher 放到第二个 watcher 的位置），导致 `$$digestOnce` 在那一轮中跳过对第三个 watcher 的执行。

> **译者注**：在上面的单元测试中，若未修正代码，这个 digest 一共会执行了三轮，结果是 `['first', 'second', 'first', 'third', 'first', 'third']`。下面来分析一下，为什么会输出这样的结果。首先，第一轮必定变“脏”，移除第二个 watcher 之后，跳过了第三个 watch 函数的执行，那这时的 $$lastDirtyWatch 会是第二个 watcher；第二轮，由于移除了第二个 watcher，此时遍历的第二个元素（也是最后一个元素）会是原本的第三个 watcher 肯定不会与 $$lastDirtyWatch（原本的第二个 watch）相等，此时第三个 watcher 第一次执行，所以第二轮 digest 也变“脏”了，会再执行第三轮 digest，因此出现这个结果。

解决这个问题的诀窍在于要对 `$$watchers` 数组进行反向操作，新注册的 watcher 会被添加到数组的开头，然后再按照从后到前的顺序进行遍历。当在 digest 的过程中有 watcher 被移除时，已经执行的 watcher 就会填满空出来的数组空间，这样不会对剩余的 watcher 产生影响。

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

然后在遍历的时候，我们需要使用 `_.forEachRight` 方法来代替 `_.forEach` 方法，改变遍历的顺序：

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

另外一个特殊情况会在 watcher 中尝试移除另一个 watcher 时出现。我们来看看下面的测试用例：

_test/scope\_spec.js_

```js
it('allows a $watch to destroy another during digest', function() {
  scope.aValue = 'abc';
  scope.counter = 0;

  scope.$watch(
    function(scope) {
      return scope.aValue;
    },
    function(newValue, oldValue, scope) {
      destroyWatch();
    }
  );

  var destroyWatch = scope.$watch(
    function(scope) {},
    function(newValue, oldValue, scope) {}
  );

  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(1);
});
```

这个单元测试没有通过，问题出在我们之前实现的短路优化上。先来回忆一下，在 `$$digestOnce` 中我们会检查当前 watcher 是否与上轮发现的最后变“脏”的 watcher 相同，如果是的话，就表明当前 digest 轮次的所有 watcher 都是“干净”（没有发生变化）的了，可以停止 digest 了。而在这个单元测试中的情况是这样的：

1. 首先，程序执行了第一个 watcher。这个 watcher 变脏了，所以 `$$lastDirtyWatch` 会被赋值为第一个 watcher，然后再调用它的 listener 函数，这时，listener 函数会销毁第二个 watcher。
2. 在第二个 watcher 被销毁时，第一个 watcher 需要向前补位，因此第一个 watcher 会被再次执行，这时它的值没有发生变化，程序比较后发现第一个 watcher 和 `$$lastDirtyWatch` 相同，满足短路优化条件，导致 digest 直接结束，第三个 watcher 也就不会被执行。

要想避免这种情况，我们需要在移除 watcher 的同时取消短路优化：

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  // var self = this;
  // var watcher = {
  //   watchFn: watchFn,
  //   listenerFn: listenerFn,
  //   valueEq: !!valueEq,
  //   last: initWatchVal
  // };
  // self.$$watchers.unshift(watcher);
  // this.$$lastDirtyWatch = null;
  // return function() {
  //   var index = self.$$watchers.indexOf(watcher);
  //   if (index >= 0) {
  //     self.$$watchers.splice(index, 1);
      self.$$lastDirtyWatch = null;
  //   }
  // };
};
```

最后我们要考虑的是一个 watcher 移除多个 watcher 的情况：

```js
it('allows destroying several $watches during digest', function() {
  scope.aValue = 'abc';
  scope.counter = 0;

  var destroyWatch1 = scope.$watch(
    function(scope) {
      destroyWatch1();
      destroyWatch2();
    }
  );

  var destroyWatch2 = scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      scope.counter++;
    }
  );

  scope.$digest();
  expect(scope.counter).toBe(0);
});
```

在这个用例中，第一个 watcher 不仅会销毁自身，还会把尚未执行的第二个 watcher 也销毁掉。这样会导致程序抛出异常（遍历第二个元素时由于元素被移除，所以 watcher 变量是 undefined，使用 undefined 执行函数自然会报错），尽管在这种情况下我们不需要执行第二个 watcher，但并不希望会抛出异常。

我们需要做的是在遍历时检查一下当前 watcher 是否真的存在：

```js
Scope.prototype.$$digestOnce = function() {
  // var self = this;
  // var newValue, oldValue, dirty;
  // _.forEachRight(this.$$watchers, function(watcher) {
    // try {
      if (watcher) {
        // newValue = watcher.watchFn(self);
        // oldValue = watcher.last;
        // if (!self.$$areEqual(newValue, oldValue, watcher.valueEq)) {
        //   self.$$lastDirtyWatch = watcher;
        //   watcher.last = (watcher.valueEq ? _.cloneDeep(newValue) : newValue);
        //   watcher.listenerFn(newValue,
        //     (oldValue === initWatchVal ? newValue : oldValue),
        //     self);
        //   dirty = true;
        // } else if (self.$$lastDirtyWatch === watcher) {
        //   return false;
        // }
      }
  //   } catch (e) {
  //     console.error(e);
  //   }
  // });
  // return dirty;
};
```

现在，无论如何移除 watcher，我们都不需要再担心会影响到 digest 的正常运行了。

