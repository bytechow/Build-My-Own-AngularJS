在之前的章节中，我们已经实现了 Angular 作用域的大部分功能了，包括 watcher 和 digest 循环。在 `Scopes` 部分的最后一个章节，我们将会实现作用域事件系统，完成最后一块拼图。

我们会发现，要实现的事件系统实际上与 digest 体系没什么关系，你可以认为是 Scope 对象提供了两个相互独立的功能。在作用域上建立事件体系之所以有用，就是在于作用域层次所构成的树结构。源自根作用域的作用域树会形成一个层次结构，并融入到每个 Angular 应用中，这种层次结构也成为了事件传递的天然渠道。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter4-watching-collections)