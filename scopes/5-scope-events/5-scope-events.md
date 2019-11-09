在之前的章节中，我们已经把 Angular 作用域的大部分功能都实现了，包括 watcher 和 digest 循环。这部分的最后一个章节，我们将要实现作用域事件系统，完成最后一块拼图。

你将会看到，事件系统实际上与 digest 系统没什么关系，因此可以说 scope 对象提供了两个不相关的功能。在作用域上实现事件系统有用的原因就在于作用域层次结构。从根作用域延展出来的作用域树结构，融入在每一个 Angular 应用程序中。这使得在树结构中传递事件成为一种天然的通信渠道。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter4-watching-collections)