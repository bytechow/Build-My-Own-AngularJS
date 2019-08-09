截止目前，差不多所有的 Angular 指令都是以数据绑定作为开始的，使用用类似`{{this}}`的模板将 JavaScript 数值绑定到 HTML 的例子。虽然我们已经在本书中讲到了变化监测和表达式，但还没介绍过这些特性的详细使用情况。这类模板语言被称为 _interpolation_（插值），每一位 Angular 开发者都对它十分熟悉了。

一个插值表达式应该包含两个双花括号，`{{`和`}}`，还有它们之间的 Angular 表达式。这个表达式其实就是我们本书第2部分介绍的那种表达式。interpolation 是一种让我们能让我们把表达式结果值加入到 DOM 的简便方式，而且当这个表达式的值发生变化时，DOM 也能随之进行变动。

> `interpolation`这个词汇来自一个在很多编程语言中都存在的特性——[string interpolation](https://en.wikipedia.org/wiki/String_interpolation)（字符串插入）。它指的是把指定占位符变成实际的数值的过程，这也是它在 Angular 要做的事。

interpolation 是由本书已经开发的功能特性的基础上实现的：Watchers，表达式和指令。本章我们会综合使用这几个特性来实现这个高级功能特性。

> 下载[本章初识代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter21-directive-transclusion)



