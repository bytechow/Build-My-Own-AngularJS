在第 1 章，我们实现了两种监测 watch 变化的策略：基于引用和基于值。在调用 `$watch` 时传入不同的布尔值标识选择使用这两种不同的方式。

本章要实现的是第三种，也是最后一种识别 watcher 变化的策略，通过一个独立函数—— `$watchCollection`。

`$watchCollection` 的使用场景是这样的：我们希望当数组或对象添加、移除或重新排序了一个元素或属性时，就会被识别为发生了变化。

正如我们在第一章讲到的，我们可以通过在调用 `$watch` 时传入 `true` 函数来注册一个基于值的 watcher 来实现这个功能。但这种策略在应对这个使用场景时所做的工作量比实际需要的多得多。这种识别策略会对从 watch 函数返回值中可以访问到的_整个对象图_（whole object graph）进行“深度侦听”。它不仅能发现元素的增减变化，也能发现元素内含元素的变化，无论这个内含元素处于哪个层次。这意味着既要保存旧值的完整深度副本，还要在 digest 中对任意深度的变化进行检查。

`$watchCollection` 其实就是对我们已经实现的基于值的 `$watch` 的优化。由于它只会对集合的最浅层进行侦听，它可以实现一个比完整深度侦听更快的、使用更少内存的检查过程。

本章都是跟 `$watchCollection` 有关的。虽然概念比较简单，但这个函数中包含了很多东西。知道了它的运作方式之后，你就能更充分地使用它了。实现 `$watchCollection` 的过程也可以作为我们对某类特定数据结构的一个学习案例，这是我们在构建 Angular 应用时需要用到的。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter3-scope-inheritance)

![](/assets/4-watching-collections/watching-collections.png)

