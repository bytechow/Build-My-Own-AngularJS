我们现在已经基于 `$watch` 和 `$digest` 构建了一个基本的脏值检测系统，但离实现作用域的全部功能还差得远呢。

本章中，我们会添加几个可以对表达式进行求值的方法，这些方法同时能触发脏值检测。我们还会学习到如何开发 `$watchGroup` 方法，这个方法能让我们同时侦听多个表达式。

> 下载[本章初识代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter1-scopes-and-digest)