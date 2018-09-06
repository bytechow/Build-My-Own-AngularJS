当前实现的注射器已经能像 Angular 注射器一般注入各种类型的值了。本章接下来的部分将会介绍如何创建能被注入的应用组件。

我们当前能在注射器中添加的仅仅是常量（constant），但这更像是直接把某个值放到了注射器的缓存中。

本章我们会学习 provider。provider 就是一个包含创建依赖的方法的对象，它可以让我们在创建依赖之前执行一些前置代码，也可以在依赖有其它前置依赖时，先加载好前置依赖。

provider 是除了 constant 之外其它所有应用组件的基础。之后介绍的 service、factory 和 value 等服务本质上都是在使用 provider。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter11-modules-and-the-injector)



