### 异常处理（Handling Exceptions）

现在我们开发的脏值检测系统跟 Angular 的越来越像了。但现在这个系统目前还是十分脆弱的，主要是因为我们还没做好异常处理。

如果在 watch 函数中发生了异常，按照目前的代码实现，结果就是停止当前要执行的任务，无论任务是什么，并且不再往下执行。但 Angular 真正的代码实现比这健壮得多，当程序出现异常时，会在 digest 期间抛出异常并记录错误日志，然后从出错的下一个 watcher 开始继续执行。

> Angular 实际上是把异常处理交由一个叫 `$exceptionHandler` 的服务来处理。本书并没有实现这项服务，因此我们会把异常直接输出到控制台。

 watcher 中有两个地方可能会发生异常：watch 函数内部和 listener 函数内部。无论是这两个地方之中的任何一处发生异常，我们希望在记录下这个异常的日志后，继续执行下一个 watcher，就算没事发生过一样。我们分别为可能出现异常的两处地方建立单元测试：

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