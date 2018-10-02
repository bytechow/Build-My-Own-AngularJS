### 使用 finally 进行清理

传统上，同步的 try...catch 语法允许我们在最后加入 finally 代码块，无论在程序执行期间是否出错，finally 代码块无论如何都会执行。Promise 也有相似的结构，同样是通过提供 finally 的方法。

这个方法将会接收一个回调函数，这个函数可以让我们在异步处理流程中执行一些清理工作。首先，这个函数会在 Promise 产生决议的时候调用，注意这个函数不会接收任何参数：

```js
it('invokes a finally handler when fulfilled', function() {
  var d = $q.defer();
  
  var finallySpy = jasmine.createSpy();
  d.promise.fnally(finallySpy);
  d.resolve(42);
  $rootScope.$apply();
  
  expect(finallySpy).toHaveBeenCalledWith();
});
```

> 我们通过没有加入参数的 toHaveBeenCalledWith 调用，来检查 finally 回调是否在调用时真的不带上参数

finally 同样会在 Promise 遭到拒绝（reject）时进行调用：

```js
it('invokes a finally handler when rejected', function() {
    var d = $q.defer();

    var finallySpy = jasmine.createSpy();
    d.promise.finally(finallySpy);
    d.reject(42);
    $rootScope.$apply();

    expect(finallySpy).toHaveBeenCalledWith();
});
```

就像 catch 一样，我们可以基于 then 实现 finally。我们会为它注册成功回调和失败回调，然后在这两个回调中执行 finally 回调，也就是说，无论如何 finally 回调都会被执行：

```js
Promise.prototype.fnally = function(callback) {
  return this.then(function() {
    callback();
  }, function() {
    callback();
  });
};
```

在后面的章节，我们会令 finally 在 Promise 处理链中也起作用。在此之前，我们先看看 Promise 链是怎么实现。