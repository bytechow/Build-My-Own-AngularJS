### 侦听对象属性的方法：$watch 和 $digest（Watching Object Properties: $watch And $digest）

`$watch` 和 `$digest`是一个硬币的两面。它们共同组成了 digest 循环的核心：对数据变化作出反应。

有了 `$watch` 你可以把一个叫 _watcher_ 的东西添加到作用域上。当作用域发生了变化时，watcher 会得到通知。我们可以通过调用 `$watch` 时传入两个参数来创建一个 watcher，而这两个参数都必须为函数：

- 一个是 watch 函数，它指定了我们要侦听其变化的数据。
- 另一个是 listener 函数，当要侦听的数据发生变化时，就会调用这个函数。

> 实际上，Angular 使用者更普遍的做法是指定侦听数据时用的是一个 watch 的表达式，而不是 watch 函数。