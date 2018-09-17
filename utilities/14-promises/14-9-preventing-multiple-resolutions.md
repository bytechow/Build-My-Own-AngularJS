### 避免多次决议（Preventing Multiple Resolutions）

Deferred 的一个重要特性是它们只会被 resolve 一次。一旦 Deferred 经过 resolve 获得一个结果值，这个值之后就不会再变更  
。Promise 的回调也只会被调用一次。如果你尝试再次对 Deferred 进行 resolve，回调调用将会被忽略：

```js
it('may only be resolved once', function() {
  var d = $q.defer();
  var promiseSpy = jasmine.createSpy();
  d.promise.then(promiseSpy);
  d.resolve(42);
  d.resolve(43);
  $rootScope.$apply();
  expect(promiseSpy.calls.count()).toEqual(1);
  expect(promiseSpy).toHaveBeenCalledWith(42);
});
```

即使两次调用之间还进行了一次 digest，这个 Deferred 的 resolve 结果值依然是不变的：

```js
it('may only ever be resolved once', function() {
  var d = $q.defer();
  var promiseSpy = jasmine.createSpy();
  d.promise.then(promiseSpy);
  d.resolve(42);
  $rootScope.$apply();
  expect(promiseSpy).toHaveBeenCalledWith(42);
  d.resolve(43);
  $rootScope.$apply();
  expect(promiseSpy.calls.count()).toEqual(1);
});
```

我们通过为内部变量 $$state 增加一个 status 属性来对 resolve 情况进行记录，一旦 Deferred 被 resolve，其对应的 Promise status 会被设置为1。在下一次调用 resolve 方法时，我们就可以进行拦截并直接跳过执行：

```js
Deferred.prototype.resolve = function(value) {
  if (this.promise.$$state.status) {
    return;
  }
  this.promise.$$state.value = value;
  this.promise.$$state.status = 1;
  scheduleProcessQueue(this.promise.$$state);
};
```



