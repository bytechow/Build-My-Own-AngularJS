### 创建 Deferred（Creating Deferreds）

我们首先要创建的并不是 Promise，而是与它密切相关的概念——Deferred。

如果 Promise 是代表某个结果值在将来的某个时刻被计算得出（处于可用状态）， Deferred 就是对结果值的运算。它们一般都是成对出现，但通常都处于代码逻辑的不同位置

