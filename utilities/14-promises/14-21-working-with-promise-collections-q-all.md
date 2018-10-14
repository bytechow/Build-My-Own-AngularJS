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