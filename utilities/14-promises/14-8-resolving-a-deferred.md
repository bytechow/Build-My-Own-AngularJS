### 解决 Deferred（Resolving A Deferred）

现在我们已经知道怎么创建 Deferred 和 Promise 了，下面将介绍他们在异步调用中的真正用处。

当我们创建了 Deferred 和 Promise 后，我们就可以为 Promise 添加一个回调函数。然后，当产生结果值时，我们就可以通过 Deferred 把结果值传递到 Promise。在之后的某个时刻，之前注册在 Promise 的回调函数就会以该值作为参数进行调用。我们可以使用 Jasmine 的 spy 函数来测试这一流程：

```js
it('can resolve a promise', function(done) {
  var deferred = $q.defer();
  var promise = deferred.promise;

  var promiseSpy = jasmine.createSpy();
  promise.then(promiseSpy);

  deferred.resolve('a-ok');

  setTimeout(function() {
    expect(promiseSpy).toHaveBeenCalledWith('a-ok');
    done();
  }, 1);
});
```

上文提到的“在之后的某个时刻”还是比较模糊，随着之后的解析，我们会对其概念更清晰。现在，我们先让测试用例通过了再说。

Promise 会把它的内部状态存储到一个叫 $$state 的内部对象变量中：

```js
function Promise() {
  this.$$state = {};
}
```

当 Promise 的 then 方法被调用后，then 的参数，也就是回调函数（callback）会被存储到 $$state 的 pending 属性中：

```js
function Promise() {
  this.$$state = {};
}
Promise.prototype.then = function(onFulflled) {
  this.$$state.pending = onFulflled;
};
```

然后，当对应的 Deferred 获取到了计算结果，我们会调用 Promise 中保存的回调函数：

```js
function Deferred() {
  this.promise = new Promise();
}
Deferred.prototype.resolve = function(value) {
  this.promise.$$state.pending(value);
};
```

这样我们就可以满足测试用例的要求了，但对于我们真正的应用需要来，这还是太过于简单了。比如，当前这个版本对于处理流程有着严格的要求。对于结果值已经计算出来，我们应该要支持