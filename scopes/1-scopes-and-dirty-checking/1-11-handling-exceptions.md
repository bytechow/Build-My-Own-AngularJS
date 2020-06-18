### 异常处理（Handling Exceptions）

现在我们实现的脏值检测系统跟正版 Angular 的越来越像了。但现在这个系统的健壮性不高，主要是因为我们还没做异常处理。

按照目前的代码实现，如果在 watch 函数中发生了异常，无论当前正在执行什么任务都会被停止，并且不再往下执行。正版 Angular 的代码可比这个健壮得多。在 digest 期间抛出的异常都会被捕获并记录日志，然后继续执行下一个 watcher。

> Angular 实际上会把异常处理交由一个叫 `$exceptionHandler` 的服务来处理。本书并没有实现这个服务，因此我们会把异常直接输出到控制台。

 watcher 中有两个地方可能会发生异常：watch 函数内部和 listener 函数内部。无论是在这两个地方中的任何一处发生异常，我们都希望记录日志，然后继续执行下一个 watcher。我们为可能出现异常的两处地方都分别建一个单元测试：

_test/scope_spec.js_

```js
it('catches exceptions in watch functions and continues', function() {
  scope.aValue = 'abc';
  scope.counter = 0;

  scope.$watch(
    function(scope) { throw 'Error'; },
    function(newValue, oldValue, scope) { }
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

it('catches exceptions in listener functions and continues', function() {
  scope.aValue = 'abc';
  scope.counter = 0;

  scope.$watch(
    function(scope) { return scope.aValue; },
    function(newValue, oldValue, scope) {
      throw 'Error';
    }
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

在这两个单元测试中，我们都注册了两个 watcher，第一个 watcher 将会抛出异常。我们只需验证第二个 watcher 有没有正常运行就可以。

要让这两个单元测试通过，我们需要修改 `$$digestOnce` 函数，把每个 watcher 的执行部分用 `try...watch` 语句包裹起来：

_src/scope.js_

```js
Scope.prototype.$$digestOnce = function() {
  // var self = this;
  // var newValue, oldValue, dirty;
  // _.forEach(this.$$watchers, function(watcher) {
    try {
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
    } catch (e) {
      console.error(e);
    }
  // });
  // return dirty;
};
```

这样一来，我们的 digest 周期就健壮多了，有异常发生也能更好地应对了！