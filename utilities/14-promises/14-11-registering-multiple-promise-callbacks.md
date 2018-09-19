### 注册多个 Promise 回调（Registering Multiple Promise Callbacks）

虽然一个 Deferred 只可以对应一个 Promise，一个 Promise 可以拥有多个回调。现在的代码还未对此进行支持：

```js
it('may have multiple callbacks', function() {
  var d = $q.defer();

  var frstSpy = jasmine.createSpy();
  var secondSpy = jasmine.createSpy();
  d.promise.then(frstSpy);
  d.promise.then(secondSpy);

  d.resolve(42);
  $rootScope.$apply();

  expect(frstSpy).toHaveBeenCalledWith(42);
  expect(secondSpy).toHaveBeenCalledWith(42);
});
```

要满足该特性，我们需要在 digest 运行且 Promise 已经被 resolve 的情况下，对所有与该 Promise 对应的还没有被调用的 callback 进行调用。但正如我们刚才提到的，每个回调应该只会被调用一次：

```js
it('invokes each callback once', function() {
  var d = $q.defer();

  var frstSpy = jasmine.createSpy();
  var secondSpy = jasmine.createSpy();

  d.promise.then(frstSpy);
  d.resolve(42);
  $rootScope.$apply();
  expect(frstSpy.calls.count()).toBe(1);
  expect(secondSpy.calls.count()).toBe(0);

  d.promise.then(secondSpy);
  expect(frstSpy.calls.count()).toBe(1);
  expect(secondSpy.calls.count()).toBe(0);
  
  $rootScope.$apply();
  expect(frstSpy.calls.count()).toBe(1);
  expect(secondSpy.calls.count()).toBe(1);
});
```



