现在我们自己实现的 Angular 依赖注入的功能已经比较完善了，也已经接近了“原版”。但它的 API 还是比较简陋，provider 尽管很强大，但还是对应用开发者不大友好。另外，目前还没办法实现在运行时进行配置的功能。

本章我们将利用已经实现的 injector 和 provider，提供更多友好的特性给应用开发者。这些特性包括更多注册（不同类型的）依赖的简便方法、一些可以配置和使用 injector 的方法。最终我们将会拥有一个真正完整的 Angular 依赖注入框架。

在本章的开发过程中，我们也需要用到一种在 JavaScript 中没有进行原生支持的数据结构—— HashMap，Angular 集成了对这一数据结构的支持，所以我们也会进行实现。

另外，为了使用依赖注入统一管理 Angular 的其余特性，我们也要用依赖注入和模块化的方法，对前三章实现的功能进行重构。

> 下载[本章初始代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter12-providers)



