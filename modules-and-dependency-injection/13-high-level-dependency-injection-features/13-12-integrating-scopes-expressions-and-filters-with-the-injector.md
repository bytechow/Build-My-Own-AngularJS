### 使用注射器注入Scope、Expression和Filter（Integrating Scopes, Expression, and Filters with The Injection）

要应用依赖注入，我们需要回顾之前章节中我们已经实现了的功能，并且使用模块加载和依赖注入两个新功能将其封装起来。这个过程是非常重要的，否则应用开发将无法应用这些特性。

在第 9 章的开始，我们已经看到了 angular 全局变量和模块加载器是如果通过 setupModuleLoader 方法创建的。现在我们要在这个方法的基础上，加入表达式解析器、根 scope、filter 服务和 filter 过滤器。

> 在本书的最后，我们会完成 Angular 完整的启动流程。

我们会将核心组件的注册放到一个新文件中，新文件命名为 src/angular\_public.js。下面我们会在里面加入第一个测试用例：

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

test/angular\_public\_spec.js

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

src/angular\_public.js

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

我们要在 filter.js 源码中默认暴露一个 provider 构造函数，这个构造函数就是我们 $filter 服务的 provider。

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

接着，我们需要修改对应的测试文件 filter\_spec.js，我们应该使用我们在上边注册的 ng 模块，而不是直接调用引入的 register 和filter:

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

filter 服务函数也可能会注入依赖：

```js
it('may have dependencies in factory', function() {
  var injector = createInjector(['ng', function($provide, $flterProvider) {
    $provide.constant('suffx', '!');
    $flterProvider.register('my', function(suffx) {
      return function(v) {
        return suffx + v;
      };
    });
  }]);
  expect(injector.has('myFilter')).toBe(true);
})
```

我们会通过在 $FilterProvider 注册过滤器的同时，在 $provider 服务也注册来解决这个问题：

```js
function $FilterProvider($provide) {
  var flters = {};
  this.register = function(name, factory) {
    if (_.isObject(name)) {
      return _.map(name, function(factory, name) {
         return register(name, factory);
      });
    } else {
        return $provide.factory(name + 'Filter', factory);
    }
  };
  this.$get = function() {
    return function flter(name) {
      return flters[name];
    };
  };
  this.register('flter', require('./flter_flter'));
}
$FilterProvider.$inject = ['$provide'];
module.exports = $FilterProvider;
```

在运行时，$filter 服务使用 $injector 获取 filter。我们不再需要内部变量 filters 来存储注册的过滤器了，也就是说，我们会使用依赖注入系统进行过滤器的存取：

```js
function $FilterProvider($provide) {
  this.register = function(name, factory) {
    if (_.isObject(name)) {
      return _.map(name, function(factory, name) {
        return register(name, factory);
      });
    } else {
      return $provide.factory(name + 'Filter', factory);
    }
  };
  this.$get = ['$injector', function($injector) {
    return function flter(name) {
      return $injector.get(name + 'Filter');
    };
  }];
  this.register('flter', require('./flter_flter'));
}
$FilterProvider.$inject = ['$provide'];
module.exports = $FilterProvider;
```

Angular 还提供了一种通过公共 API 注册过滤器的快捷方式。它会在模块实例上挂载一个 filter 方法，该方法的第一个参数是过滤器的名称，第二个参数是过滤器函数。它比在 config 服务中使用 $filterProvider 注册过滤器要更方便：

```js
it('can be registered through module API', function() {
  var myFilter = function() {};
  var module = window.angular.module('myModule', [])
    .flter('my', function() {
      return myFilter;
    });
  var injector = createInjector(['ng', 'myModule']);
  expect(injector.has('myFilter')).toBe(true);
  expect(injector.get('myFilter')).toBe(myFilter);
});
```

下面，在模块加载器，我们将会引入 filter 方法：

```js
var moduleInstance = {
  name: name,
  requires: requires,
  constant: invokeLater('$provide', 'constant', 'unshift'),
  provider: invokeLater('$provide', 'provider'),
  factory: invokeLater('$provide', 'factory'),
  value: invokeLater('$provide', 'value'),
  service: invokeLater('$provide', 'service'),
  decorator: invokeLater('$provide', 'decorator'),
  flter: invokeLater('$flterProvider', 'register'),
  confg: invokeLater('$injector', 'invoke', 'push', confgBlocks),
  run: function(fn) {
    moduleInstance._runBlocks.push(fn);
    return moduleInstance;
  },
  _invokeQueue: invokeQueue,
  _confgBlocks: confgBlocks,
  _runBlocks: []
};
```

跟其他注册方法不同的是，filter 方法不需要考虑 $provide 方法的调用顺序，而会在 $filterProvider 中进行调用顺序的重排序。

我们也会需要更新我们的 filter 过滤器：

```js
'use strict';
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');
describe('flter flter', function() {
  beforeEach(function() {
    publishExternalAPI();
  });
  it('is available', function() {
    var injector = createInjector(['ng']);
    expect(injector.has('flterFilter')).toBe(true);
  });
  // ...
});
```

> 注意，当前 filter\_filter\_spec.js 文件的其他用例依然会报错，这是因为我们还没有重构 parse 服务

接下来，我们要重构的是表达式解析器（parser）:

```js
it('sets up the $parse service', function() {
  publishExternalAPI();
  var injector = createInjector(['ng']);
  expect(injector.has('$parse')).toBe(true);
});
```

它被注册为一个 provider：

```js
function publishExternalAPI() {
  setupModuleLoader(window);
  var ngModule = angular.module('ng', []);
  ngModule.provider('$flter', require('./flter'));
  ngModule.provider('$parse', require('./parse'));
}
```

跟 filter 一样，我们也希望在 parse.js 里面有一个 provider 构造函数，这个构造函数可以用语构建 $parse 服务的 provider。

```js
function $ParseProvider() {
  this.$get = function() {
    return function(expr) {
      switch (typeof expr) {
        case 'string':
          var lexer = new Lexer();
          var parser = new Parser(lexer);
          var oneTime = false;
          if (expr.charAt(0) === ':' && expr.charAt(1) === ':') {
            oneTime = true;
            expr = expr.substring(2);
          }
          var parseFn = parser.parse(expr);
          if (parseFn.constant) {
            parseFn.$$watchDelegate = constantWatchDelegate;
          } else if (oneTime) {
            parseFn.$$watchDelegate = parseFn.literal ?
              oneTimeLiteralWatchDelegate :
              oneTimeWatchDelegate;
          } else if (parseFn.inputs) {
            parseFn.$$watchDelegate = inputsWatchDelegate;
          }
          return parseFn;
        case 'function':
          return expr;
        default:
          return _.noop;
      }
    };
  };
}
module.exports = $ParseProvider;
```

实际上就是把之前的 parse 函数作为 $get 方法的返回值，也就是说，我们注入的 $parse 变量就是 parse 函数。

我们还需要往 parse 中获取 $filter 服务。首先，我们要移除 require filter 的代码。然后，我们要采用依赖注入的方式，往 $get 方法中注入 $filter 服务

```js
this.$get = ['$flter', function($flter) {
  return function(expr) {
    switch (typeof expr) {
      case 'string':
        var lexer = new Lexer();
        var parser = new Parser(lexer, $flter);
        var oneTime = false;
        if (expr.charAt(0) === ':' && expr.charAt(1) === ':') {
          oneTime = true;
          expr = expr.substring(2);
        }
        var parseFn = parser.parse(expr);
        if (parseFn.constant) {
          parseFn.$$watchDelegate = constantWatchDelegate;
        } else if (oneTime) {
          parseFn.$$watchDelegate = parseFn.literal ? oneTimeLiteralWatchDelegate :
            oneTimeWatchDelegate;
        } else if (parseFn.inputs) {
          parseFn.$$watchDelegate = inputsWatchDelegate;
        }
        return parseFn;
      case 'function':
        return expr;
      default:
        return _.noop;
    }
  };
}];
```

Parser 构造函数则会传递 $filter 服务给 ASTCompiler 构造函数：

```js
function Parser(lexer, $flter) {
  this.lexer = lexer;
  this.ast = new AST(this.lexer);
  this.astCompiler = new ASTCompiler(this.ast, $flter);
}
```

ASTCompiler 构造函数会把它保存为自己的实例属性：

```js
function ASTCompiler(astBuilder, $flter) {
  this.astBuilder = astBuilder;
  this.$flter = $flter;
}
```

第一处会用到这个实例属性的地方是生成表达式代码时可能会用到的 filter 方法，我们实际上会使用 $filter 服务进行过滤：

```js
/* jshint -W054 */
var fn = new Function(
  'ensureSafeMemberName',
  'ensureSafeObject',
  'ensureSafeFunction',
  'ifDefned',
  'flter',
  fnString)(
  ensureSafeMemberName,
  ensureSafeObject,
  ensureSafeFunction,
  ifDefned,
  this.$flter);
/* jshint +W054 */
```

第二处用到 $filter 的地方是 markConstantAndWatchExpressions，这个函数之前使用的是全局的 filter 函数：

```js
ASTCompiler.prototype.compile = function(text) {
  var ast = this.astBuilder.ast(text);
  var extra = '';
  markConstantAndWatchExpressions(ast, this.$flter);
  // ...
};
```

在 markConstantAndWatchExpressions 中，由于会进行递归调用，我们也需要在所有递归调用处使用 $filter 服务：

```js
function markConstantAndWatchExpressions(ast, $flter) {
  var allConstants;
  var argsToWatch;
  switch (ast.type) {
    case AST.Program:
      allConstants = true;
      _.forEach(ast.body, function(expr) {
        markConstantAndWatchExpressions(expr, $flter);
        allConstants = allConstants && expr.constant;
      });
      ast.constant = allConstants;
      break;
    case AST.Literal:
      ast.constant = true;
      ast.toWatch = [];
      break;
    case AST.Identifer:
      ast.constant = false;
      ast.toWatch = [ast];
      break;
    case AST.ArrayExpression:
      allConstants = true;
      argsToWatch = [];
      _.forEach(ast.elements, function(element) {
        markConstantAndWatchExpressions(element, $flter);
        allConstants = allConstants && element.constant;
        if (!element.constant) {
          argsToWatch.push.apply(argsToWatch, element.toWatch);
        }
      });
      ast.constant = allConstants;
      ast.toWatch = argsToWatch;
      break;
    case AST.ObjectExpression:
      allConstants = true;
      argsToWatch = [];
      _.forEach(ast.properties, function(property) {
        markConstantAndWatchExpressions(property.value, $flter);
        allConstants = allConstants && property.value.constant;
        if (!property.value.constant) {
          argsToWatch.push.apply(argsToWatch, property.value.toWatch);
        }
      });
      ast.constant = allConstants;
      ast.toWatch = argsToWatch;
      break;
    case AST.ThisExpression:
      ast.constant = false;
      ast.toWatch = [];
      break;
    case AST.MemberExpression:
      markConstantAndWatchExpressions(ast.object, $flter);
      if (ast.computed) {
        markConstantAndWatchExpressions(ast.property, $flter);
      }
      ast.constant = ast.object.constant &&
        (!ast.computed || ast.property.constant);
      ast.toWatch = [ast];
      break;
    case AST.CallExpression:
      var stateless = ast.flter && !$flter(ast.callee.name).$stateful;
      allConstants = stateless ? true : false;
      argsToWatch = [];
      _.forEach(ast.arguments, function(arg) {
        markConstantAndWatchExpressions(arg, $flter);
        allConstants = allConstants && arg.constant;
        if (!arg.constant) {
          argsToWatch.push.apply(argsToWatch, arg.toWatch);
        }
      });
      ast.constant = allConstants;
      ast.toWatch = stateless ? argsToWatch : [ast];
      break;
    case AST.AssignmentExpression:
      markConstantAndWatchExpressions(ast.left, $flter);
      markConstantAndWatchExpressions(ast.right, $flter);
      ast.constant = ast.left.constant && ast.right.constant;
      ast.toWatch = [ast];
      break

    case AST.UnaryExpression:
      markConstantAndWatchExpressions(ast.argument, $flter);
      ast.constant = ast.argument.constant;
      ast.toWatch = ast.argument.toWatch;
      break;
    case AST.BinaryExpression:
      markConstantAndWatchExpressions(ast.left, $flter);
      markConstantAndWatchExpressions(ast.right, $flter);
      ast.constant = ast.left.constant && ast.right.constant;
      ast.toWatch = ast.left.toWatch.concat(ast.right.toWatch);
      break;
    case AST.LogicalExpression:
      markConstantAndWatchExpressions(ast.left, $flter);
      markConstantAndWatchExpressions(ast.right, $flter);
      ast.constant = ast.left.constant && ast.right.constant;
      ast.toWatch = [ast];
      break;
    case AST.ConditionalExpression:
      markConstantAndWatchExpressions(ast.test, $flter);
      markConstantAndWatchExpressions(ast.consequent, $flter);
      markConstantAndWatchExpressions(ast.alternate, $flter);
      ast.constant =
        ast.test.constant && ast.consequent.constant && ast.alternate.constant;
      ast.toWatch = [ast];
      break;
  }
}
```

现在我们可以满足刚才加到 parse\_spec.js 文件里单元测试了，但是其他测试单元还是报错了，这是因为它们还是依赖之前的全局函数 parse。因此，我们需要修改 parse\_spec.js 中对 parse 服务的引入，我们会在 beforeEach 代码块中创建一个 injector 来获取 $parse 服务，代码如下：

```js
'use strict';
var _ = require('lodash');
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');
describe('parse', function() {
  var parse;
  beforeEach(function() {
    publishExternalAPI();
    parse = createInjector(['ng']).get('$parse');
  });
  // ...
});
```

那些注册和使用过滤器的测试单元也需要进行升级，它们也会创建自己的注射器，并通过 $filterProvider 注册过滤器：

```js
it('can parse flter expressions', function() {
  parse = createInjector(['ng', function($flterProvider) {
    $flterProvider.register('upcase', function() {
      return function(str) {
        return str.toUpperCase();
      };
    });
  }]).get('$parse');
  var fn = parse('aString | upcase');
  expect(fn({
    aString: 'Hello'
  })).toEqual('HELLO');
});
it('can parse flter chain expressions', function() {
  parse = createInjector(['ng', function($flterProvider) {
    $flterProvider.register('upcase', function() {
      return function(s) {
        return s.toUpperCase();
      };
    });
    $flterProvider.register('exclamate', function() {
      return function(s) {
        return s + '!';
      };
    });
  }]).get('$parse');
  var fn = parse('"hello" | upcase | exclamate');
  expect(fn()).toEqual('HELLO!');
});
it('can pass an additional argument to flters', function() {
  parse = createInjector(['ng', function($flterProvider) {
    $flterProvider.register('repeat', function() {
      return function(s, times) {
        return _.repeat(s, times);
      };
    });
  }]).get('$parse');
  var fn = parse('"hello" | repeat:3');
  expect(fn()).toEqual('hellohellohello');
});
it('can pass several additional arguments to flters', function() {
  parse = createInjector(['ng', function($flterProvider) {
    $flterProvider.register('surround', function() {
      return function(s, left, right) {
        return left + s + right;
      };
    });
  }]).get('$parse');
  var fn = parse('"hello" | surround:"*":"!"');
  expect(fn()).toEqual('*hello!');
});
// ...
it('marks flters constant if arguments are', function() {
  parse = createInjector(['ng', function($flterProvider) {
    $flterProvider.register('aFilter', function() {
      return _.identity;
    });
  }]).get('$parse');
  expect(parse('[1, 2, 3] | aFilter').constant).toBe(true);
  expect(parse('[1, 2, a] | aFilter').constant).toBe(false);
  expect(parse('[1, 2, 3] | aFilter:42').constant).toBe(true);
  expect(parse('[1, 2, 3] | aFilter:a').constant).toBe(false);
});
```

现在，我们也可以用同样的方法对 filter 过滤器进行处理：

```js
'use strict';
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');
describe('flter flter', function() {
  var parse;
  beforeEach(function() {
    publishExternalAPI();
    parse = createInjector(['ng']).get('$parse');
  });
  // ...
});
```

目前，$parse 服务已经被成功注入，让我们把目光转向 scope.js。scope 的相关单元测试已经有部分在报错了，这是因为它们依赖的全局函数 parse 和 register 已经不复存在了。接下来，我们来快速修复一下。

就像 $filter 和 $parse 服务，ng 模块需要注入一个 scope 实例作为根 scope，也就是 $rootScope：

```js
it('sets up the $rootScope', function() {
  publishExternalAPI();
  var injector = createInjector(['ng']);
  expect(injector.has('$rootScope')).toBe(true);
});
```

同样使用 provider 注册 $rootScope：

```js
function publishExternalAPI() {
  setupModuleLoader(window);
  var ngModule = angular.module('ng', []);
  ngModule.provider('$flter', require('./flter'));
  ngModule.provider('$parse', require('./parse'));
  ngModule.provider('$rootScope', require('./scope'));
}
```

我们要做的就是把 scope.js 的实现代码都放到 $RootScopeProvider.$get 的方法体中：

```js
'use strict';
var _ = require('lodash');

function $RootScopeProvider() {
  this.$get = function() {
    // Move all previous code from scope.js here.
  };
}
module.exports = $RootScopeProvider;
```

我们还需要从 $get 方法中返回一个值，所以我们最后会创建并返回一个 Scope 实例：

```js
function $RootScopeProvider() {
  this.$get = function() {
    // All previous code from scope.js goes here.
    var $rootScope = new Scope();
    return $rootScope;
  };
}
module.exports = $RootScopeProvider;
```

显然，这个返回值就是 $rootScope。注意，目前所有 scope 相关的属性和变量已变成了私有的了，无法在 provider 外部被访问到。我们对外暴露的只有 $rootScope。

当然，我们要把之前依赖 parse 全局函数的地方改成使用 $parse 服务。由于我们已经创建了 provider 和它的 $get 方法，我们就可以注入 $parse 服务：

```js
'use strict';
var _ = require('lodash');

function $RootScopeProvider() {
  this.$get = ['$parse', function($parse) {
    // All previous code from scope.js goes here.
    var $rootScope = new Scope();
    return $rootScope;
  }];
}

module.exports = $RootScopeProvider;
```

现在我们可以改变 $watch 函数，将它的依赖从 parse 全局函数变成 $parse 服务：

```js
Scope.prototype.$watch = function(watchFn, listenerFn, valueEq) {
  // ...
  watchFn = $parse(watchFn);
  // ...
};
```

对 $eval 方法也做同样的处理：

```js
Scope.prototype.$eval = function(expr, locals) {
  return $parse(expr)(this, locals);
};
```

就这样，我们就实现了 $rootScope。但对于 $rootScope 服务的单元测试还存在问题。接下来，我们会继续进行修复。

首先得更新 scope\_spec.js 最前面的引用依赖的代码。现在，我们只需要模块和注射器就可以了：

```js
'use strict';

var _ = require('lodash');
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');
```

接着，我们遗憾底宣布，scope_spec.js 的第一个测试用例将会被去掉，就是下面这段代码：

```js
it('can be constructed and used as an object', function() {
  var scope = new Scope();
  scope.aProperty = 1;
  expect(scope.aProperty).toBe(1);
});
```

现在，我们要获得一个 Scope，需要通过 $rootScope 服务，而不是构造函数。所以我们要去除这个测试用例。你可以通过 $rootScope 获取一个 scope，这个我们已经在 angular_public_spec.js 注册并测试了。

现在我们需要检查 scope_spec.js 中所有嵌套的 describe 代码块，并且使用 injector 加载一个 scope 对象，以便现存的测试用例使用。

在 scope_spec.js 每个测试模块，如 describe('digest')，describe('$eval')，describe('$apply')，describe('$evalAsync')，describe('$applyAsync')，describe('$$postDigest') 和 describe('$watchGroup')，我们会为它们新增一个 beforeEach 代码块，在代码块里面我们会通过注射器获取根作用域。