现在我们已经有了一个可以表示字面量的表达式语言，但这在实际场景中的用处不大。Angular 表达式语言本就是针对访问作用域数据这个目的进行设计的，同时也可能会对数据进行操作。

本章我们会对上述功能进行扩展。我们会学习几种访问属性的方式，还有如何调用函数。我们也会开发一些安全措施来防止在 Angular 表达式语言中编写危险表达式。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter6-literal-expressions)