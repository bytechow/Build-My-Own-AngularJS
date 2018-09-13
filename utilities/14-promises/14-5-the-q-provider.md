### $q Provider（The $q provider）

本章我们会完整实现 $q 服务，从中我们会清晰地看到 Promise 在 Angular 中是如何构建起来的的。首先，我们还是要先做一些准备工作。我们应该保证 $q 服务集成中 ng 模块中：

```js
it('sets up $q', function() {
  publishExternalAPI();
  var injector = createInjector(['ng']);
  expect(injector.has('$q')).toBe(true);
});
```

要让 $q 服务可用，我们需要使用 provider 来对服务进行封装，就跟之前的 $parse 和 $rootScope 一样。

我们将会新建一个 q.js 文件，在里面我们存放 $q 服务对应的 provider。

src/q.js

```js
'use strict';

function $QProvider() {

  this.$get = function() {
    //...
  };

}

module.exports = $QProvider;
```

当然要通过测试，我们还需要在 ng 模块在注册 $q 服务：

```js
function publishExternalAPI() {
  setupModuleLoader(window);

  var ngModule = angular.module('ng', []);
  ngModule.provider('$filter', require('./filter'));
  ngModule.provider('$parse', require('./parse'));
  ngModule.provider('$rootScope', require('./scope'));
  ngModule.provider('$q', require('./q'));
}
```