### 创建 Deferred（Creating Deferreds）

我们首先要创建的并不是 Promise，而是与它密切相关的概念——Deferred。

如果 Promise 是代表某个结果值在将来的某个时刻被计算得出（处于可用状态）， Deferred 就是对结果值的运算。它们一般都是成对出现，但通常都处于代码逻辑的不同位置。

从数据流的角度来看，数据生产者持有 Deferred，而数据消费者持有 Promise。当数据生产者完成了 Deferred 计算，数据消费者就会通过 Promise 接收到结果值。 

![](/assets/deferred-and-promise.png)

> ECMAScript 6 标准定义的 Promise 并没有 Deferred 的概念，它使用普通函数来处理运算的生产者部分。AngularJS 也支持这种方式，我们会在本章结尾看到它的具体应用方式。但使用 Deferred 来实现依然是更为基础的、被广泛使用的方式（截止到本书写作期间）

我们

