###  即时拒绝-$q.reject（Immediate Rejection - $q.reject）

当我们需要写一个返回 Promise 的异步函数时，有时会清楚结果很可能是失败，或者根本就希望获取一个失败的结果。在这种情况下，我们可能更希望直接返回一个已经被拒绝的 Promise，而不需要再进行异步操作。实际上，依靠现在的代码，我们就可以实现了：

```js
var d = $q.defer();
d.reject('fail');
return d.promise;
```

但这种方式还是有点啰嗦了，因此 $q 服务提供了一种简便的方式——reject：

```js
it('can make an immediately rejected promise', function() {
  var fulflledSpy = jasmine.createSpy();
  var rejectedSpy = jasmine.createSpy();

  var promise = $q.reject('fail');
  promise.then(fulflledSpy, rejectedSpy);

  $rootScope.$apply();

  expect(fulflledSpy).not.toHaveBeenCalled();
  expect(rejectedSpy).toHaveBeenCalledWith('fail');
});
```

reject 的实现代码正正就是上面那一段代码而已：

```js
function reject(rejection) {
  var d = defer();
  d.reject(rejection);
  return d.promise;
}
```

剩下的，就是把 reject 方法对外暴露而已：

```js
return {
  defer: defer,
  reject: reject
};
```