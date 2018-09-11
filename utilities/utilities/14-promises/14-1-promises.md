### Promises（约定）

Promises 就是为了解决上述问题而诞生出来的技术。它的原理就是把一个异步操作的结果打包到一个对象中去。一个异步函数，返回的不再是一个具体值，而是一个将在未来某个时刻获得结果值的 promise。当然，promise 也提供了让你访问结果值的接口，以便在获取到结果值时执行后续处理。

> promise 有时候获取到的结果值就是 undefined，这样的结果值很容易让我们产生误解，以为函数根本没有返回值。但即使是这样，我们也能够知道这个结果值就是异步计算的结果，这能够起到一定作用。

Promise 的机制让我们可以摆脱传统回调函数需要在业务流程函数中附加参数的麻烦，这个回调函数实际上已经转移到函数的返回值—— promise 中去了：

```js
computeBalance(from, to).then(function(balance) {
  // ...
});
```

Promise 提供的链式操作也解决了传统回调函数可能出现“回调地狱”的问题。注册一个 Promise 回调，同时也会返回一个新的 Promise，这给了我们一个更加扁平的控制流：

```js
computeBalance(a, b)
  .then(storeBalance)
  .then(displayResults);
```

当然 Promise 也提供了 API 让我们可以处理异步操作中发生的错误：

```js
computeBalance(a, b)
  .then(storeBalance)
  .catch(handleError);
```

Promise 处理流中发生的错误，将会顺着 Promise 处理链进行冒泡，最终进入 catch 处理块进行处理。是的，十分类似 try...catch 处理的逻辑，而且 Promise 也支持多个成功回调函数：

```js
computeBalance(a, b)
  .then(storeBalance)
  .then(displayResults)
  .catch(handleError);
```



