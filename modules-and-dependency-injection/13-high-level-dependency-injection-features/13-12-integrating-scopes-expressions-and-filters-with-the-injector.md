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



