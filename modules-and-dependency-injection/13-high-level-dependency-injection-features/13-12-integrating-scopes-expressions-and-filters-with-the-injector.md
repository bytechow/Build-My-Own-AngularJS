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

$filter 服务将会被注册为一个 provider：

```js
function publishExternalAPI(){
  setupModuleLoader(window)

  var ngModule = angular.module('ng', [])
  ngModule.provider('$filter', require('./filter'))
}
```

改造如此，也就说我们要在 filter.js 源码中默认暴露一个 provider 构造函数，这个构造函数就是我们 $filter 服务的 provider。

在本书的第二部分，我们在 filter.js 文件中做的仅仅是建立和对外暴露两个函数——register 和 filter。现在我们需要把它们封装到 provider 中了。这个 register 函数将会变成 provider 的一个方法，而 filter 函数将会变成 $get 方法的返回值。换句话说，filter 函数将会成为我们在应用中使用的 $filter 服务：

```js
function $FilterProvider() {
  var filters = {};
  this.register = function(name, factory) {
    if (_.isObject(name)) {
      return _.map(name, _.bind(function(factory, name) {
        return this.register(name, factory);
      }, this));
    } else {
      var filter = factory();
      filters[name] = filter;
      return filter;
    }
  };
  this.$get = function() {
    return function filter(name) {
      return filters[name];
    };
  };
  this.register('filter', require('./filter_filter'));
}
module.exports = $FilterProvider;
```

接着，我们需要修改对应的测试文件 filter_spec.js，我们应该使用我们在上边注册的 ng 模块，而不是直接调用引入的 register 和filter:

```js
'use strict';
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');
describe('filter', function() {
  beforeEach(function() {
    publishExternalAPI();
  });

  it('can be registered and obtained', function() {
    var myFilter = function() {};
    var myFilterFactory = function() {
      return myFilter;
    };
    var injector = createInjector(['ng', function($filterProvider) {
      $filterProvider.register('my', myFilterFactory);
    }]);
    var $filter = injector.get('$filter');
    expect($filter('my')).toBe(myFilter);
  });

  it('allows registering multiple filters with an object', function() {
    var myFilter = function() {};
    var myOtherFilter = function() {};
    var injector = createInjector(['ng', function($filterProvider) {
      $filterProvider.register({
        my: function() {
          return myFilter;
        },
        myOther: function() {
          return myOtherFilter;
        }
      });
    }]);
    var $filter = injector.get('$filter');
    expect($filter('my')).toBe(myFilter);
    expect($filter('myOther')).toBe(myOtherFilter);
  });
});
```

另外，$filter 服务将会对 filter 进行实例化，并对其进行依赖注入，让它们成为常规函数。如果你注册了一个过滤器名为 my，它除了在 $filter 服务中可用，也会生成一个名为 myFilter 的依赖在缓存中：

```js
it('is available through injector', function() {
  var myFilter = function() {};
  var injector = createInjector(['ng', function($filterProvider) {
    $filterProvider.register('my', function() {
      return myFilter;
    });
  }]);
  expect(injector.has('myFilter')).toBe(true);
  expect(injector.get('myFilter')).toBe(myFilter);
});
```