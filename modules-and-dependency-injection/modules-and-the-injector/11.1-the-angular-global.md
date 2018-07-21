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



