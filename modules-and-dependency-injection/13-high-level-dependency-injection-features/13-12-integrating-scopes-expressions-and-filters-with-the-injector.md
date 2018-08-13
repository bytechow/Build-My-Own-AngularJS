### 使用注射器注入Scope、Expression和Filter（Integrating Scopes, Expression, and Filters with The Injection）

要应用依赖注入，我们需要回顾之前章节中我们已经实现了的功能，并且使用模块加载和依赖注入两个新功能将其封装起来。这个过程是非常重要的，否则应用开发将无法应用这些特性。

在第 9 章的开始，我们已经看到了 angular 全局变量和模块加载器是如果通过 setupModuleLoader 方法创建的。现在我们要在这个方法的基础上，加入表达式解析器、根 scope、filter 服务和 filter 过滤器。

> 在本书的最后，我们会完成 Angular 完整的启动流程。

我们会将核心组件的注册放到一个新文件中，新文件命名为 src/angular_public.js。下面我们会在里面加入第一个测试用例：

```js
'use strict';
var publishExternalAPI = require('../src/angular_public');
describe('angularPublic', function() {
  it('sets up the angular object and the module loader', function() {
    publishExternalAPI();
    expect(window.angular).toBeDefned();
    expect(window.angular.module).toBeDefned();
  });
});
```

这个用例非常简单，只是把之前的 setupModuleLoader 放到了 publishExternalAPI 中执行而已：

```js
'use strict';
var setupModuleLoader = require('./loader');

function publishExternalAPI() {
  setupModuleLoader(window);
}
module.exports = publishExternalAPI;
```

这个函数应该初始化 ng 模块：

test/angular_public_spec.js

```js
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');
describe('angularPublic', function() {
  it('sets up the angular object and the module loader', function() {
    publishExternalAPI();
    expect(window.angular).toBeDefned();
    expect(window.angular.module).toBeDefned();
  });
  it('sets up the ng module', function() {
    publishExternalAPI();
    expect(createInjector(['ng'])).toBeDefned();
  });
});
```

src/angular_public.js

```js
function publishExternalAPI() {
  setupModuleLoader(window);
  var ngModule = window.angular.module('ng', []);
}
```

这个 ng 模块将会承载所有的 Angular 原生组件，包括 service、directive、filter 等组件。之后，我们学到 Angular 启动（bootstrap）后，我们会在这个模块中存放所有的 Angular 应用。对于应用开发者来说，他们也不需要知道它的存在。但这就是 Angular 暴露自己的服务给其他服务或者应用的方式。

> 由于我们正在使用依赖注入的方式重构之前实现的功能，所以暂时所有的测试用例可能都会受影响，不必担心，等我们完全处理好之后，所有的测试用例都会恢复正常。

首先要放到 ng 模块里面去的是 $filter，也就是我们的过滤器服务：

```js
it('sets up the $flter service', function() {
  publishExternalAPI();
  var injector = createInjector(['ng']);
  expect(injector.has('$flter')).toBe(true);
});


```