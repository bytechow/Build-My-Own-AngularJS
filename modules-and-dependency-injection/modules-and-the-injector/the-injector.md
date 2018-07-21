### 注射器（The Injector）

接下来我们将开始实现 Angular 依赖注入的另一个特性——注射器。

注射器并不是模块加载器的一部分，是一个独立的服务，所以我们需要新建一个源代码文件和测试文件。同样地，在测试用例中，我们需要重置模块加载器（防止之前用例产生的缓存影响）。

test/injector\_spec.js

```js
'use strict';
var setupModuleLoader = require('../src/loader');
describe('injector', function() {
  beforeEach(function() {
    delete window.angular;
    setupModuleLoader(window);
  });
});
```

要创建注射器，我们可以通过传入模块名称数组作为参数调用函数 createInjector：

test/injector\_spec.js

```js
'use strict';
var setupModuleLoader = require('../src/loader');
var createInjector = require('../src/injector');
describe('injector', function() {
  beforeEach(function() {
    delete window.angular;
    setupModuleLoader(window);
  });
  it('can be created', function() {
    var injector = createInjector([]);
    expect(injector).toBeDefined();
  });
});
```

现在为了简单起见，我们直接返回一个空对象字面量：

src/injector.js

```js
'use strict';

function createInjector(modulesToLoad) {
return {};
}
module.exports = createInjector;
```



