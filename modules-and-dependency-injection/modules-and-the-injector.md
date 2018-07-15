这个章节将会搭建Angular依赖注入的框架。我们会介绍两个重要的特性——模块（modules）与注射器（injector），同时介绍如何利用这两个特性来注册应用组件，并把应用组件注入到需要它们的地方中去。

在这两个特性中，模块是大部分应用开发者都会直接使用到的。简单来说，模块就是应用配置信息的集合。我们需要通过模块来注册服务（services）、控制器（controllers）、指令（directives）、过滤器（filters）等应用组件。

但注射器才是能让应用运作起来的幕后功臣。只有当你创建一个注射器，并利用它来实例化模块，模块依赖的应用组件才会被创建注入（若已经创建过，则直接注入），注入完成以后，模块才能真正发挥作用。

本章节，我们将学习如何创建模块，然后学习如何创建一个注射器来加载这些模块

> 下载[本章代码](https://github.com/teropa/build-your-own-angularjs/releases/tag/chapter10-expressions-and-watches)

### 全局对象 angular（The angular Global）

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

这个测试假定 loader.js 中存在一个函数 setupModuleLoader，如果调用该函数并传入window对象作为参数，则会生成一个全局对象 angular

接下来就是创建 loader.js ，并让测试通过。

src/loader.js

```js
'use strict';

function setupModuleLoader(window) {
  var angular = window.angular = {};
}
module.exports = setupModuleLoader;
```

### 只实例化一次全局对象（Initializing The Global Just Once）

angular 全局对象存储已注册的模块，其本质上也是全局状态的储存器。这意味着我们需要找到管理状态的方法。首先，我们希望单元测试之间互不干扰，这需要在每个单元测试开始之前删除已存在的 angular 全局对象：

```js
beforeEach(function(){
  delete window.angular;
})
```

另外，在 setupModuleLoader 函数中，我们需要保证 angular 全局对象不会被覆盖，即使我们调用了多次 setupModuleLoader 函数。在测试中，当我们调用 setupModuleLoader 两次时，第一次调用后和第二次调用后访问 angular 全局对象，其结果应该指向同一个对象：

```js
it('creates angular just once', function() {
  setupModuleLoader(window);
  var ng = window.angular;
  setupModuleLoader(window);
  expect(window.angular).toBe(ng);
});
```

我们可以通过简单的检测来解决这个问题：

```js
function setupModuleLoader(window) {
  var angular = (window.angular = window.angular || {});
}
```

之后很快我们就再一次使用这种“单例模式”，所以我们先抽象出一个通用的函数 ensure，这个函数将会接收三个参数：一个对象 obj，对象的属性名 name 和一个“”工厂函数“ factory。当对象 obj.name 不存在，才会调用工厂函数产生一个值，并赋值到obj.name：

```js
function setupModuleLoader(window) {
  var ensure = function(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
  };
  var angular = ensure(window, 'angular', Object);
}
```

这里我们先用 Object 构造函数生成一个空对象，赋值给 angular 全局对象。注意Object\( \) 与 new Object\( \) 在效果上一样的。

### module 方法（The module Method）

我们要介绍的第一个在 angular 全局对象的方法是 module，这个方法将会在本章以及之后的章节被大量使用。首先，我们断然这个方法存在与新创建的 angular 全局对象

test/loader\_spec.js

```js
it('exposes the angular module function', function() {
  setupModuleLoader(window);
  expect(window.angular.module).toBeDefined();
});
```

就像全局对象 angular 一样，module 方法也应该是单例的：

```js
it('exposes the angular module function just once', function() {
  setupModuleLoader(window);
  var module = window.angular.module;
  setupModuleLoader(window);
  expect(window.angular.module).toBe(module);
})
```

现在我们可以重用 ensure 函数：

```js
function setupModuleLoader(window) {
  var ensure = function(obj, name, factory) {
    return obj[name] || (obj[name] = factory());
  };
  var angular = ensure(window, 'angular', Object);
  ensure(angular, 'module', function() {
    return function() {};
  });
}
```



