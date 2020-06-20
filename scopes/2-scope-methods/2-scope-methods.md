我们通过实现 `$watch` 和 `$digest` 这两个方法，构建起了一个基本的脏值检测系统，但作用域系统本身要实现的功能还很多。

本章我们会介绍几个可以对表达式进行求值的方法，这些方法也能触发脏值检测。我们还会学习一个叫 `$watchGroup` 的方法，这个方法能让我们同时侦听多个表达式。

> 下载[本章初识代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter1-scopes-and-digest)