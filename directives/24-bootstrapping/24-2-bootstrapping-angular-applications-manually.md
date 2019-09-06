### 手动启动 Angular 应用（Bootstrapping Angular Applications Manually）

现在，让我们把注意力放到怎么真正地启动一个 AngularJS 应用上：对于 Angular 应用来说，需要做一些什么事情才能让它呈现在网页上？

我们有两种方法可以启动应用：

- 手动启动，是指开发者通过显式地调用 Angular 给予的启动 API 对应用进行启动。
- 自动启动，如果 Angular 发现 DOM 里面有`ng-app`属性， 会找到这个属性的位置，并自行启动程序。

自动启动是构建在手动启动的基础上的，所以我们先来讲一下手动启动。

我们会在新创建的文件中实现，这也意味着我们会用一个新创建的测试文件来存放对应的测试用例。我们来加入以下的一些引用：

test/bootstrap_spec.js

```js
'use strict';

var $ = require('jquery');
var bootstrap = require('../src/bootstrap');

describe('bootstrap', function() {

});
```

我们可以通过调用`angular.bootstrap()`函数进行应用的手动启动。我们先来测试一下，看`angular`全局对象中是否含有这个函数：

```js
describe('manual', function() {

  it('is available', function() {
    expect(window.angular.bootstrap).toBeDefined();
  }); 

});
```

注意，这里我们不需要再做任何设置工作了，因为引入`bootstrap`模块的行为本身就会定义`window.angular`全局变量，而`bootstrap`方法也会加入到这个全局变量上。

我们可以在`bootstrap.js`文件中先做好引入依赖，并调用`publicExternalAPI`函数，这个函数是我们之前已经在开发依赖注入功能时做好了的。这个函数就会引入`window.angular`。之后，我们只需要把`bootstrap`添加到这个对象上就好了：

_src/bootstrap.js_

```js
'use strict';

var $ = require('jquery');
var publishExternalAPI = require('./angular_public');

publishExternalAPI();

window.angular.bootstrap = function() {
};
```

那么，bootstrap 方法究竟会做些什么呢？稍后我们会看到，它有几项职责。其中一项是我们需要为新应用创建一个_injector_（注射器）。这个注射器最终会成为`bootstrap`方法的返回值：

```js
it('creates and returns an injector', function() {
  var element = $('<div></div>');
  var injector = window.angular.bootstrap(element);
  expect(injector).toBeDefined();
  expect(injector.invoke).toBeDefined();
});
```

要创建一个 injector，我们需要引入已经开发好的`createInjector`函数：

```js
'use strict';

var $ = require('jquery');
var publishExternalAPI = require('./angular_public');
var createInjector = require('./injector');

publishExternalAPI();

window.angular.bootstrap = function() {
  var injector = createInjector();
  return injector;
};
```

当你要启动一个 Angular 应用是，你都是在某个 HTML 元素上进行启动的，这个元素会成为应用的根元素。&lt;body&gt;经常为成为根元素，但根元素也可以是页面上的其他元素。无论是什么元素，当你需要进行手动启动时，你都需要把这个元素给到`bootstrap`方法。之后要做的一件事就是把这个注射器以 jQuery data 的方式附加到这个元素上：

```js
it('attaches the injector to the bootstrapped element', function() {
  var element = $('<div></div>');
  var injector = window.angular.bootstrap(element);
  expect(element.data('$injector')).toBe(injector);
});
```