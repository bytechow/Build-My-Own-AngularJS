现在我们已经开发了一个完整的脏值检测系统和一个完整的 Angular 表达式语言，但我们还没有把二者结合起来。所有 Angular 使用者都知道，只有把二者结合起来，才能体现 Angular 脏值检测系统真正的强大实力。本章过后，我们将会实现二者的组合。

本章我们会把表达式和作用域进行集成，作为对前两章的总结。除此以外，我们还会加入一些强有力的优化措施：_常量侦测_（Constant detection），_单次绑定_（One-time binding）和_输入跟踪_（input tracking）。然后在结束本章之前，我们会研究表达式如何做到可以既用于求值，也用于重新赋值。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter9-filters)
