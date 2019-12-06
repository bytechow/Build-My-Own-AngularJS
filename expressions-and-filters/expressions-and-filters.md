![expressions-and-filters](/assets/expressions-and-filters.png)

我们已经完整开发了作用域系统，它同时也是 Angular 变化侦测系统的核心。下面我们把注意力转移到 _expression_ 上面来，expression 是一个可以对变化侦测系统进行“无摩擦”访问的特性。通过表达式，我们可以精准地访问和操作作用域上的数据，并在上面进行一些计算。

我们可以在 JavaScript 应用代码中使用表达式，效果也很好，但这不是表达式最主要的应用场景。表达式的真正价值在于允许我们把数据和行为绑定到 HTML 模板上。我们会在指令属性比如 `ngClass` 和 `ngClick` 上使用到表达式，