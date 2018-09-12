### AngularJS 中的 Promise（Promises in AngularJS）

AngularJS 也实现了 Promise，主要通过 $q 服务。顾名思义，AngularJS 的 promise 实现灵感来自于 Q ，你可以把 AngularJS Promise 看作是 Q 的简化版。

$q 服务同样符合 Promises/A+ 标准，因此我们也可以使用类似 ES6 Promise 的用法来使用它，之后会详细介绍。

$q 服务与其他 Promise 实现方案的一个重要区别是 $q 将会与 digest 循环集成在一起。$q 中的一切变更都会发生在 Angular digest 中，所以我们也无需考虑 scope.$apply 调用的问题。另外，很多的库使用 setTimeout 来形成异步，而 $q 服务用之前实现的 $evalAsync 就可以了。

