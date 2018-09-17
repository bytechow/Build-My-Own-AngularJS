### 保证回调被调用（Ensuring that Callbacks Get Invoked）

目前，我们会在 Deferred 被 resolve 后延时执行 Promise 回调。看似没有问题，但其实我们忽略了一种情况。如果 Promise 回调是在 Deferred 已经被 resolve，且已经有一个 digest 运行过，这个回调是不会被执行的：

```js
it('resolves a listener added after resolution', function() {
  var d = $q.defer();
  d.resolve(42);
  $rootScope.$apply();
  var promiseSpy = jasmine.createSpy();
  d.promise.then(promiseSpy);
  $rootScope.$apply();
  expect(promiseSpy).toHaveBeenCalledWith(42);
});
```

为了解决这个问题，我们需要在注册回调时对 status 进行检查，如果 Deferred 已经被 resolve，我们就再次设置延时执行回调

```js
Promise.prototype.then = function(onFulflled) {
  this.$$state.pending = onFulflled;
  if (this.$$state.status > 0) {
    scheduleProcessQueue(this.$$state);
  }
};
```