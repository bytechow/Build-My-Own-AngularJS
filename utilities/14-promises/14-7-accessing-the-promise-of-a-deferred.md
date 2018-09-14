### 获取 Deferred 对应的 Promise（Accessing The Promise of A Deferred）

Deferred 和 Promise 始终都是成对出现。创建一个 Deferred，也意味有一个 Promise 被生成了。我们可以通过 Deferred 实例的 promise 属性访问其对应的 Promise：

```js
it('has a promise for each Deferred', function() {
  var d = $q.defer();
  expect(d.promise).toBeDefned();
});
```

在 $q 服务中，Promise 同样有一个同名构造函数。Deferred 构造函数会调用 Promise 构造函数来获取一个新的 Promise：

```js
this.$get = function() {
  function Promise() {}

  function Deferred() {
    this.promise = new Promise();
  }

  function defer() {
    return new Deferred();
  }
  return {
    defer: defer
  };
};
```



