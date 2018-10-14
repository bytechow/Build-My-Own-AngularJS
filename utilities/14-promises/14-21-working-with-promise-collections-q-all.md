### 处理 Promise 集合——$q.all（Working with Promise Collections）

当有多个异步任务要执行时，将它们打包成一个 Promise 集合会变得很有用。Promise 说白了就是一个 JavaScript 对象而已，所以实际上你可以用任何可以生成、操作和转换集合的函数或库对它们进行处理。

如果有专门针对 Promise的集合方法，这些方法可以组合并处理异步任务，在某些情况下会更有用。社区已经有相关的解决方案了，比如[async-q]https://www.npmjs.com/package/async-q