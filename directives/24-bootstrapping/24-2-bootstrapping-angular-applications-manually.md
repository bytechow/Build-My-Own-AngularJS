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

Angular 本身并不需要用这个数据属性做些什么，这个属性只是用于使用工具和调试的目的。

我们会确保传入的元素是一个 jQuery 对象，然后对它加入一些数据属性：

_src/bootstrap.js_

```js
window.angular.bootstrap = function(element) {
  var $element = $(element);
  var injector = createInjector();
  $element.data('$injector', injector);
  return injector;
};
```

现在我们的注射器中还什么都没有，没有注入依赖的话我们无法做出一个有用的 Angular 应用！我们至少应该把所有 Angular 内建的服务都注入进来，比如`$compile`和`$rootScope`：

```js
it('loads built-ins into the injector', function() {
  var element = $('<div></div>');
  var injector = window.angular.bootstrap(element);

  expect(injector.has('$compile')).toBe(true);
  expect(injector.has('$rootScope')).toBe(true);
});
```

我们要确保`ng`模块加入到了注射器中。这样我们就把`angular_public.js`中所有内建的服务都注入进来了：

```js
window.angular.bootstrap = function(element) {
  var $element = $(element);
  var injector = createInjector(['ng']);
  $element.data('$injector', injector);
  return injector;
};
```

但这还不够。用户想要启动的是他们自己的应用，这些应用会有用户自定义的模块组成。添加的这些模块会作为`bootstrap`方法的第二个参数。这个参数是一个由模块名称组成的数组，这些模块之前都应该通过`angular.module`注册好了：

_test/bootstrap_spec.js_

```js
it('loads other specified modules into the injector', function() {
  var element = $('<div></div>');

  window.angular.module('myModule', [])
    .constant('aValue', 42);
  window.angular.module('mySecondModule', [])
    .constant('aSecondValue', 43);
  window.angular.bootstrap(element, ['myModule', 'mySecondModule']);
  
  var injector = element.data('$injector');
  expect(injector.get('aValue')).toBe(42);
  expect(injector.get('aSecondValue')).toBe(43);
});
```

开发时，我们会在给定的数组的最前面插入`ng`模块，这样内建的服务总是会先被加载：

_src/bootstrap.js_

```js
window.angular.bootstrap = function(element, modules) {
  var $element = $(element);
  modules = modules || [];
  modules.unshift('ng');
  var injector = createInjector(modules);
  $element.data('$injector', injector);
  return injector;
};
```

除了把所有在模块上定义的依赖，`bootstrap`方法也会支持注入一个特殊的依赖项` $rootElement`。它让应用开发者能访问到应用的 DOM 根元素：

_test/bootstrap_spec.js_

```js
it('makes root element available for injection', function() {
  var element = $('<div></div>');

  window.angular.bootstrap(element);
  
  var injector = element.data('$injector');
  expect(injector.has('$rootElement')).toBe(true);
  expect(injector.get('$rootElement')[0]).toBe(element[0]);
});
```

我们可以通过注册一个函数形式的模块来完成这个功能。我们创建这个模块，然乎把它放到`modules`数组的前面就可以了。这个模块会注册一个叫`$rootElement`的 value 依赖，它会指向我们要在其上启动应用的元素：

_src/bootstrap.js_

```js
window.angular.bootstrap = function(element, modules) {
  var $element = $(element);
  modules = modules || [];
  modules.unshift(['$provide', function($provide) {
    $provide.value('$rootElement', $element);
  }]);
  modules.unshift('ng');
  var injector = createInjector(modules);
  $element.data('$injector', injector);
  return injector;
};
```

这些依赖注入实际的设置过程就无需我们关心了。`bootstrap`方法的第二个也是非常重要的职责就是对启动元素包含的 DOM 进行编译。这意味着在这个元素底下的任何指令都会进行编译：

_test/bootstrap_spec.js_

```js
it('compiles the element', function() {
  var element = $('<div><div my-directive></div></div>');
  var compileSpy = jasmine.createSpy();
  
  window.angular.module('myModule', [])
    .directive('myDirective', function() {
      return {compile: compileSpy};
    });
  window.angular.bootstrap(element, ['myModule']);

  expect(compileSpy).toHaveBeenCalled();
});
```

我们会通过注射器获取到`$compile`服务，然后对启动元素进行使用。我们可以使用`$injector.invoke()`在注入`$compile`服务的同时进行编译：

_src/bootstrap.js_

```js
window.angular.bootstrap = function(element, modules) {
  // var $element = $(element);
  // modules = modules || [];
  // modules.unshift(['$provide', function($provide) {
  //   $provide.value('$rootElement', $element);
  // }]);
  // modules.unshift('ng');
  // var injector = createInjector(modules);
  // $element.data('$injector', injector);
  injector.invoke(['$compile', function($compile) {
    $compile($element);
  }]);
  // return injector;
};
```

实际上，在启动过程中，DOM 元素不仅会进行编译，也会被链接到应用的根作用域上：

_test/bootstrap_spec.js_

```js
it('links the element', function() {
  var element = $('<div><div my-directive></div></div>');
  var linkSpy = jasmine.createSpy();

  window.angular.module('myModule', [])
    .directive('myDirective', function() {
      return {link: linkSpy};
    });
  window.angular.bootstrap(element, ['myModule']);

  expect(linkSpy).toHaveBeenCalled();
  expect(linkSpy.calls.mostRecent().args[0]).toEqual(
    element.data('$injector').get('$rootScope')
  );
});
```

我们可以把`$rootScope`注入到`invoke`函数中去，然后在编译后马上运行这个公共链接函数：

_src/bootstrap.js_

```js
window.angular.bootstrap = function(element, modules) {
  // var $element = $(element);
  // modules = modules || [];
  // modules.unshift(['$provide', function($provide) {
  //   $provide.value('$rootElement', $element);
  // }]);
  // modules.unshift('ng');
  // var injector = createInjector(modules);
  // $element.data('$injector', injector);
  injector.invoke(['$compile', '$rootScope', function($compile, $rootScope) {
    $compile($element)($rootScope);
  }]);
  // return injector;
};
```

同时，这意味着在此刻应用初始的 scope 继承层次已经建立起来了：任何一个使用继承作用域或独立作用域的指令出现，都会随之产生一个新的作用域。这些工作都由编译器完成，`bootstrap`方法并不需要对此做什么特殊的处理。

第三个，也是`bootstrap`的最重要的一个职责是启动应用的第一个 digest。这一步会在初始的 DOM 已经编译、链接好后进行。这也意味着初始 DOM 中的插值表达式会被表达式的结果值替换掉：

_test/bootstrap_spec.js_

```js
it('runs a digest', function() {
  var element = $('<div><div my-directive>{{expr}}</div></div>');
  var linkSpy = jasmine.createSpy();

  window.angular.module('myModule', [])
    .directive('myDirective', function() {
      return {
        link: function(scope) {
          scope.expr = '42';
        }
      }
    });
  window.angular.bootstrap(element, ['myModule']);

  expect(element.find('div').text()).toBe('42');
});
````

我们能做的就是把我们的编译和链接操作放到`$rootScope.$apply()`调用内部就好了，这样就能确保在链接完成以后会运行一个 digest：

_src/bootstrap.js_

```js
window.angular.bootstrap = function(element, modules) {
  // var $element = $(element);
  // modules = modules || [];
  // modules.unshift(['$provide', function($provide) {
  //   $provide.value('$rootElement', $element);
  // }]);
  // modules.unshift('ng');
  // var injector = createInjector(modules);
  // $element.data('$injector', injector);
  injector.invoke(['$compile', '$rootScope', function($compile, $rootScope) {
    $rootScope.$apply(function() {
      $compile($element)($rootScope);
    });
  }]);
  // return injector;
};
```

`bootstrap`还支持一个特性，就是启用严格的依赖注入模式，启用严格模式的代码我们已经在书中的依赖注入板块实现了。启用严格的注入模式后，不是通过数组包裹或使用`$inject`进行注解的注入方式都会抛出异常。要是在指令中使用了这样的一个依赖注入，而我们使用了`bootstrap`的第三个参数来启用严格的注入模式，程序应该运行失败：

_test/bootstrap_spec.js_

```js
it('supports enabling strictDi mode', function() {
  var element = $('<div><div my-directive></div></div>');
  var compileSpy = jasmine.createSpy();

  window.angular.module('myModule', [])
    .constant('aValue', 42)
    .directive('myDirective', function(aValue) {
      return {};
    });
  
  expect(function() {
    window.angular.bootstrap(element, ['myModule'], { strictDi: true });
  }).toThrow();
});
```

`bootstrap`方法的第三个参数实际上是一个配置对象，但目前只支持一个 key：`strictDi`。我们会把这个参数传递给`createInjector`函数进行调用，作为这个函数的第二个参数——`strictDi`：

_src/bootstrap.js_

```js
window.angular.bootstrap = function(element, modules, config) {
  // var $element = $(element);
  // modules = modules || [];
  config = config || {};
  // modules.unshift(['$provide', function($provide) {
  //   $provide.value('$rootElement', $element);
  // }]);
  // modules.unshift('ng');
  var injector = createInjector(modules, config.strictDi);
  // $element.data('$injector', injector);
  // injector.invoke(['$compile', '$rootScope', function($compile, $rootScope) {
  //   $rootScope.$apply(function() {
  //     $compile($element)($rootScope);
  //   }); 
  // }]);
  // return injector;
};
```

这样我们就有了完整的一个 Angular 应用启动程序了。它包括：

1. 使用内建的和自建的模块，创建一个依赖注射器。
2. 将传入的元素（应用根元素）进行编译和链接。
3. 启动应用的第一个 digest。

在这之后，应用中发生的事情就全都由指令、控制器和服务这些模块包含的内容决定了。