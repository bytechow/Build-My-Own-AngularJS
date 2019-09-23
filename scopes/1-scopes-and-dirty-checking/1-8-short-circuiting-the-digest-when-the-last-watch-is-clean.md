### 如果最后一个 watcher 未发生变化则绕过 digest（Short-Circuiting The Digest When The Last Watch Is Clean）

在目前的代码中，我们会对作用域中 watcher 集合进行持续遍历，直到在某轮遍历中所有 watcher 都未发生变化（或者达到了 TTL 上限）。

鉴于一个 digest 循环可能包含大量的 watcher，尽可能地降低执行 watcher 的次数是非常重要的。