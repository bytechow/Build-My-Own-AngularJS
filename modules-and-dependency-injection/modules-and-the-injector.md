这个章节将会搭建Angular依赖注入的框架。我们会介绍两个重要的特性——模块（modules）与注射器（injector），同时介绍如何利用这两个特性来注册应用组件，并把应用组件注入到需要它们的地方中去。

在这两个特性中，模块是大部分应用开发者都会直接使用到的。简单来说，模块就是应用配置信息的集合。我们需要通过模块来注册服务（services）、控制器（controllers）、指令（directives）、过滤器（filters）等应用组件。

但注射器才是能让应用运作起来的幕后功臣。只有当你创建一个注射器，并利用它来实例化模块，模块依赖的应用组件才会被创建注入（若已经创建过，则直接注入），注入完成以后，模块才能真正发挥作用。

本章节，我们将学习如何创建模块，然后学习如何创建一个注射器来加载这些模块

> 下载[本章代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter10-expressions-and-watches)

### The angular Global

用过Angular的人都应该接触过全局对象 angular，现在是时候引入这个对象了。

需要有容器来承载模块和注射器，而这个容器就是 angular 全局对象。

处理模块的框架组件被称为_模块加载器（module loader）_，我们会把这个组件的代码放到 loader.js 中。在 loader.js 中，我们正式引入 angular 全局对象。但首先，我们会按照惯例先创建对应的测试代码。

test/loader\_spec.js

```js
'use strict';

var setupModuleLoader = require('../src/loader');

describe('setupModuleLoader', function() {

  it('exposes angular on the window', function() {
    setupModuleLoader(window);
    expect(window.angular).toBeDefined();
  });
});
```





