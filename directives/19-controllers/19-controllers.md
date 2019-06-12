控制器在 AngularJS 是一个有趣的“野兽”。考虑到所谓的“Model-View 控制器”模式的 JavaScript 框架十分受欢迎，而 AngularJS 也被认为是这样的框架，因此控制器看上去是 Angular 中非常重要的组成部分。

实际上，控制器并不是 Angular 中最重要的角色之一。确实，控制器十分重要，在很多应用中也是如此。但它们实际上知识指令系统的一个组成部分。Angular 中的主角是指令，而控制器是配角，用于协助指令完成功能。单独的一个控制器，比如`ngController`，最多只能算是指令系统的“副产品”，下面我们就会介绍到。

但这不代表控制器就是无趣、无用的。事实恰恰相反，学习到下面我们就会知道了。控制器由三个部分组成，我们会分别对这三个部分进行介绍：`$controller` provider，指令编译器与控制器的整合、`ngController`指令。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter18-directive-linking-and-scopes)



