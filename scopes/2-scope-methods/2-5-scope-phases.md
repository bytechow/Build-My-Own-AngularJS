### 作用域阶段
#### Scope Phases

`$evalAsync`还可以在检测到当前没有 digest 运行时定时调用 `$digest` 方法。那意味着，无论你在什么时候使 `$evalAsync` 来延迟执行一个函数，你都可以保证这个函数会在“不久后”被调用，而不需要等待其他东西来启动一个 digest。

> 虽然 `$evalAsync` 会定时启动一个 `$digest`，但对于要异步执行代码的我们更推荐