### 返回 Promise 的回调函数（Callbacks Returning Promises）

现在，当一个 Promise 回调返回了一个值，我们会把这个值作为结果值传到下一个 Promise 节点中去；当 Promise 回调抛出一个异常，这个异常也会传递到下一个 Promise 节点的失败回调。这两个情景都是发生在同步处理当中的。现在，如果我们需要在 Promise 回调中做一些异步的处理，换句话说，我们需要返回的是一个新的 Promise，又该怎么办呢？

答案就是我们需要在 Promise 返回值和下一个节点回调之间建立联系：

```js
it('waits on promise returned from handler', function() {
  var d = $q.defer();
  var fulflledSpy = jasmine.createSpy();

  d.promise.then(function(v) {
    var d2 = $q.defer();
    d2.resolve(v + 1);
    return d2.promise;
  }).then(function(v) {
    return v * 2;
  }).then(fulflledSpy);

  d.resolve(20);

  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith(42);
});
```

我们使用链式调用的方式绑定了三个回调，第一个回调进行了一个“异步”的操作，只有在这个异步操作结束后，我们才会继续往下执行第二个回调。

还有另外一种在 Promise 处理链中使用异步的情况，就是使用一个 Proimse（名字设为A）作为当前 Promise（名字设为B） resolve 的参数，这样当 Promise A resolve 的时候，它的 resolution 也会成为 Promise B 的 resolution：

```js
it('waits on promise given to resolve', function() {
  var d = $q.defer();
  var d2 = $q.defer();
  var fulflledSpy = jasmine.createSpy();

  d.promise.then(fulflledSpy);
  d2.resolve(42);
  d.resolve(d2.promise);

  $rootScope.$apply();

  expect(fulflledSpy).toHaveBeenCalledWith(42);
});
```

这两条特性同样适用于 Promise 被 reject 的情况。当回调内部有已经 reject 掉的 Promise 作为返回值，这个 rejection 也在 Promise 链中进行传递：

```js
it('rejects when promise returned from handler rejects', function() {
  var d = $q.defer();
  var rejectedSpy = jasmine.createSpy();
  
  d.promise.then(function() {
    var d2 = $q.defer();
    d2.reject('fail');
    return d2.promise;
  }).catch(rejectedSpy);
  d.resolve('ok');

  $rootScope.$apply();
  
  expect(rejectedSpy).toHaveBeenCalledWith('fail');
});
```

总结起来，我们需要处理的情况是：一个 Deferred 可能使用另一个 Promise 作为 resolution，这时这个 Deferred 的 resolution 就依赖于这个 Promise 的 resolution。因此，当我们调用 resolve 方法时，我们需要检查传入的参数是不是一个 Promise。如果通过判断证明是一个 Promise，那我们就不会立即进行 resolve，相反，我们为这个 Promise 绑定了回调函数，这个回调函数还是来自外层 Promise 的，当内层 Promise 有结果后，就会再次调用外层 Promise 的 resolve（或reject）方法：

```js
Deferred.prototype.resolve = function(value) {
  if (this.promise.$$state.status) {
    return;
  }
  if (value && _.isFunction(value.then)) {
    value.then(
      _.bind(this.resolve, this),
      _.bind(this.reject, this)
    );
  } else {
    this.promise.$$state.value = value;
    this.promise.$$state.status = 1;
    scheduleProcessQueue(this.promise.$$state);
  }    
};
```

这样的话，我们所有的测试用例就都通过了，因为它们都会调用 resolve 方法来进行流程。

> 注意，在上面的代码实现中，我们没有对 Promise 进行严格的类型判定。事实上，我们并要求它必须是一个 Promise 的示例，只要是一个含有 then 方法的对象（也称为 thenable 对象），我们都是允许的。这样做的好处是，我们可以使用其他的 Promise 实现作为嵌套 Promise，也就意味着，我们可以在 AngularJS 中兼容使用其他库或者就是 ES2015 的 Promise。



