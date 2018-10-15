### 处理 Promise 集合——$q.all（Working with Promise Collections）

当有多个异步任务要执行时，将它们打包成一个 Promise 集合会变得很有用。Promise 说白了就是一个 JavaScript 对象而已，所以实际上你可以用任何可以生成、操作和转换集合的函数或库对它们进行处理。

如果有专门针对 Promise的集合方法，这些方法可以组合并处理异步任务，在某些情况下会更有用。社区已经有相关的解决方案了，比如[async-q](https://www.npmjs.com/package/async-q)。然而，Angular 已经提供了一个专门用于组合并处理 Promise 集合的方法了，它就是——$q.all。

$q.all 方法接收一个 Promise 集合作为它的参数，返回值就是集合中各个 Promise 的计算结果值。结果数组的元素与集合中的 Promise 一一对应，这对于同时进行的异步任务来说尤其有用:

```js
describe('all', function() {

  it('can resolve an array of promises to array of results', function() {
    var promise = $q.all([$q.when(1), $q.when(2), $q.when(3)]);
    var fulflledSpy = jasmine.createSpy();
    promise.then(fulflledSpy);

    $rootScope.$apply();
    
    expect(fulflledSpy).toHaveBeenCalledWith([1, 2, 3]);
  });
});
```

显然，这也是一个新的 API：

```js
function all(promises) {

}

return {
  defer: defer,
  reject: reject,
  when: when,
  resolve: when,
  all: all
};
```

这个函数会创建一个存放计算结果的数据，并为每一个 Promise 增加一个回调函数，结果数组的元素位置与 Promise 的位置一一对应：

```js
function all(promises) {
  var results = [];
  _.forEach(promises, function(promise, index) {
    promise.then(function(value) {
      results[index] = value;
    });
  });
}
```

在这个函数里面还有一个内部变量充当计数器，每一次循环都将会加1，而如果有 Promise 回调被调用就会减 1:

```js
function all(promises) {
  var results = [];
  var counter = 0;
  _.forEach(promises, function(promise, index) {
    counter++;
    promise.then(function(value) {
      results[index] = value;
      counter--;
    });
  });
}
```

这意味着如果计数器变成0，则所有的 Promise 都已经被 resolve 了。如果我们在里面构造一个 Deferred，并让它在计数器变成 0 后 resolve，就完成了 $q.all 的逻辑实现了：

```js
function all(promises) {
  var results = [];
  var counter = 0;
  var d = defer();
  _.forEach(promises, function(promise, index) {
    counter++;
    promise.then(function(value) {
      results[index] = value;
      counter--;
      if (!counter) {
        d.resolve(results);
      }
    });
  });
  return d.promise;
}
```

