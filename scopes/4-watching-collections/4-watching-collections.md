在第 1 章，我们实现了两种监测 watch 变化的策略：引用变化和值变化。在调用 `$watch` 时传入不同的布尔值标识就可以选择使用这两种不同的方式。

本章要实现的是第三种，也是最后一种识别 watcher 变化的策略，这策略通过一个独立函数—— `$watchCollection` 实现。

`$watchCollection` 的使用场景是这样的：我们希望当数组或对象添加、移除或重新排序了一个元素或属性时，就会被识别为发生了变化，。

我们在第一章说过，我们可以选择在调用 `$watch` 时传入 `true` 来注册一个基于值的 watcher，这种监测方式也可以满足上述场景，但它所需要的工作量远比实际的多。这种监测方式会对 watch 函数返回的_整个对象图_（whole object graph）进行“深度侦听”。它不仅可以发现对象元素的增减变化，也可以发现内嵌元素的变化，无论这个内嵌元素处于哪个层次。这意味着除了要保存旧值的完整深度副本， digest 还要支持对任意深度的元素变化进行检查。

`$watchCollection` 其实就是对我们已经实现的基于值的 `$watch` 的优化。这个函数只会对集合的浅层进行侦听，因而可以实现一个比完整深度侦听更快的、内存占用更少的检查过程。

本章都是跟 `$watchCollection` 有关的。虽然概念简单，但这个函数中包含了很多东西。知道了它的运作方式之后，你就可以更充分地使用它了。实现 `$watchCollection` 的过程也可以作为我们对特定数据结构的一个学习案例，这是我们在构建 Angular 应用时需要用到的。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter3-scope-inheritance)

![](/assets/4-watching-collections/watching-collections.png)

