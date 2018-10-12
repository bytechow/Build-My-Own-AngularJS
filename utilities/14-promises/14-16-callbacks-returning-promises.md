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

我们使用链式调用的方式绑定了三个回调，第一个做了一个“异步”的操作，只有在这个异步操作结束后，我们才会继续往下执行第二个回调。

