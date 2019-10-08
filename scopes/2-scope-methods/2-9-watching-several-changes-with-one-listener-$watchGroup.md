### 多个变化共用同一个 listener 函数
#### Watching Several Changes With One Listener: $watchGroup

到目前为止，我们看到所有的 watch 函数和 listener 函数是一个简单的因果关系对：当这个发生了变化，就那样做。但还有一种情况也并不少见，就是同时观察多个状态，当其中一个状态发生改变时就执行某段代码。

鉴于 Angular watch 就是一个普通的 JavaScript 函数，我们目前的 watch 代码就能完美支持这种情况了：我们可以在 watch 函数对多个要侦听的数据进行查询，最后返回一个包含所有查询结果的复合值就可以了，这个复合值发生的变化就可以触发 listener 函数了。

> 译者注：当然这里说的 watcher 需要传入第三个参数值 `true`，开启基于值的比较模式才可以检测复合值内容是否发生变化。

事实上，在 Angular 1.3 以后的版本中已经不再需要手动创建这类函数，我们可以直接使用 Angular 内建的 `$watchGroup`。


