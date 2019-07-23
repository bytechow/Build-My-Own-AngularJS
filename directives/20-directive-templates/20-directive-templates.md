添加指令到已存在的 DOM 是一种方式，但使用指令自身构造的 DOM 也是非常普遍的用法。一个指令不仅仅可以对一个已存在的元素进行装饰修改，也能够在内部生成元素内容。举例来说，如果你使用一个`<login-form>`指令，你会希望它渲染成一个带有输入框、按钮和标签的登录表单。

在某种程度上，我们目前的指令系统已经对此进行支持了，因为我们可以在指令的`compile`和`link`函数中进行任意的 DOM 操作。你可以在里面使用 JavaScript 的 DOM API 来满足指令的需求。

但这并不算方便。如果我们可以提供一种机制，可以对 HTML 字符串进行处理，这就更好了。相当于告诉 Angular：“如果应用了这个指令，就把指令里面的 HTML 内容生成出来”。实际上，这就是_指令模板_的功能，在本章我们会对这个知识点进行介绍。

> 指令模板与独立作用域结合使用时会尤其有用，它能帮助我们用“组件化”的方式构建应用 UI。在这方面，Angular 2 已经走在前头了，但我们还是可以使用 [Angular 1.x 的方式](http://teropa.info/blog/2014/10/24/how-ive-improved-my-angular-apps-by-banning-ng-controller.html)来应用这种模式

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter19-controllers)

