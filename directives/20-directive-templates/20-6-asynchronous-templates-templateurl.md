### 异步模板：templateUrl（Asynchronous Templates: templateUrl）

当使用`template`属性时，你会往里面填充一行长长的 HTML 代码字符串。但这不是存放 HTML 代码的好地方，尤其是在代码很长的情况下。

如果能把 HTML 模板放到以`.html`为后缀的文件里存放，然后再把它加载到应用中这样会方便很多。为此，Angular 提供了一个`templateUrl`属性。当定义了这个属性，模板就会通过 HTTP 加载指定 URL 的 HTML 文件。

由于通过 HTTP 加载文件总是异步进行的，这就意味着当我们遇到一个要加载 URL 的模板时，我们需要在模板加载完之前暂停编译进程。然后在模板加载进来后，我们需要继续编译。这章的剩余部分大多就是用于处理这种暂停、恢复编译的需求。

有关的第一个单元测试是当遇到需要异步加载模板的指令时，是否能够停止编译。

首先，在这种情况下，当前元素上的其他指令就不应该继续进行编译了：

_test/compile_spec.js_

```js
describe('templateUrl', function() {

  it('defers remaining directive compilation', function() {
    var otherCompileSpy = jasmine.createSpy();
    var injector = makeInjectorWithDirectives({
      myDirective: function() {
        return {
          templateUrl: '/my_directive.html'
        };
      },
      myOtherDirective: function() {
        return {
          compile: otherCompileSpy
        };
      }
    });
    injector.invoke(function($compile) {
      var el = $('<div my-directive my-other-directive></div>');
      $compile(el);
      expect(otherCompileSpy).not.toHaveBeenCalled();
    });
  });
  
});
```

现在，当我们遇到带有`templateUrl`属性的指令时，我们要做的是直接中止循环。我们可以通过返回`false`来结束循环，因为 Lodash 的`_.forEach`结束循环的方式：

_src/compile.js_

```js
// if (directive.template) {
//   if (templateDirective) {
//     throw 'Multiple directives asking for template';
//   }
//   templateDirective = directive;
//   $compileNode.html(_.isFunction(directive.template) ?
//     directive.template($compileNode, attrs) :
//     directive.template);
// }
if (directive.templateUrl) {
  return false;
}
```

不但是需要中断其他指令的编译，在加载到模板 URL 之前，当前指令也不应该进行编译：

_test/compile_spec.js_

```js
it('defers current directive compilation', function() {
  var compileSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html',
        compile: compileSpy
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive></div>');
    $compile(el);
    expect(compileSpy).not.toHaveBeenCalled();
  });
});
```

要通过这个测试，我们需要将代码进行一些移动。`if(directive.compile){ }`这个代码块需要放到进行`templateUrl`检查之后的`else if`分支。代码块里面的代码不需要改变。这样能够保证这段代码不会在指令有`templateUrl`的情况下执行：

_src/compile.js_

```js
_.forEach(directives, function(directive) {
  // if (directive.$$start) {
  //   $compileNode = groupScan(compileNode, directive.$$start, directive.$$end);
  // }
  // if (directive.priority < terminalPriority) {
  //   return false;
  // }
  // if (directive.scope) {
  //   if (_.isObject(directive.scope)) {
  //     if (newIsolateScopeDirective || newScopeDirective) {
  //       throw 'Multiple directives asking for new/inherited scope';
  //     }
  //     newIsolateScopeDirective = directive;
  //   } else {
  //     if (newIsolateScopeDirective) {
  //       throw 'Multiple directives asking for new/inherited scope';
  //     }
  //     newScopeDirective = newScopeDirective || directive;
  //   }
  // }
  // if (directive.controller) {
  //   controllerDirectives = controllerDirectives || {};
  //   controllerDirectives[directive.name] = directive;
  // }
  // if (directive.template) {
  //   if (templateDirective) {
  //     throw 'Multiple directives asking for template';
  //   }
  //   templateDirective = directive;
  //   $compileNode.html(_.isFunction(directive.template) ?
  //     directive.template($compileNode, attrs) :
  //     directive.template);
  // }
  if (directive.templateUrl) {
  //   return false;
  } else if (directive.compile) {
    var linkFn = directive.compile($compileNode, attrs);
    var isolateScope = (directive === newIsolateScopeDirective);
    var attrStart = directive.$$start;
    var attrEnd = directive.$$end;
    var require = directive.require;
    if (_.isFunction(linkFn)) {
      addLinkFns(null, linkFn, attrStart, attrEnd, isolateScope, require);
    } else if (linkFn) {
      addLinkFns(linkFn.pre, linkFn.post,
        attrStart, attrEnd, isolateScope, require);
    }
  }
  // if (directive.terminal) {
  //   terminal = true;
  //   terminalPriority = directive.priority;
  // }
});
```

另一个要做的是，这时当前元素的内容应该被清空了。元素内容会被即将载入的模板内容代替，但我们需要立即清除所有的旧内容，这样旧内容就不会被不必要地编译了：

_test/compile_spec.js_

```js
it('immediately empties out the element', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html'
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive>Hello</div>');
    $compile(el);
    expect(el.is(':empty')).toBe(true);
  });
});
```

要达到这个目标，我们需要引入一个新函数`compileTemplateUrl`，它的工作就是处理加入了模板以后的所有异步工作：

_src/compile.js_

```js
if (directive.templateUrl) {
  compileTemplateUrl($compileNode);
  return false;
}
````

这个函数（会在`applyTemplatesToNode`函数外加入）目前只需要清空节点内容，就可以让我们的测试通过：

```js
function compileTemplateUrl($compileNode) {
  $compileNode.empty();
}
```

实际上，我们目前实现的就是，当指令定义了`templateUrl`，就挂起当前 DOM 子树的编译过程。当前指令或指令所在元素上的其他指令都不会被编译，而元素的子节点也会被移除掉。

现在我们可以开始思考我们什么时候应该继续编译过程。首先需要做的事通过 URL 来获取指定的模板。我们可以利用本书上一部分中实现的`$http`服务来完成。

为了测试模板的获取，我们需要引入 Sinon.js 提供的模拟 XMLHttpRequest 的功能，我们在`$http`章节也用到了这个功能。

首先，在`compile_spec.js`中引入 Sinon：

_test/compile_spec.js_

```js
'use strict';

// var _ = require('lodash');
// var $ = require('jquery');
var sinon = require('sinon');
// var publishExternalAPI = require('../src/angular_public');
// var createInjector = require('../src/injector');
```

然后，在`describe('templateUrl')`测试板块中加下以下安装代码：

```js
describe('templateUrl', function() {
  var xhr, requests;
  
  beforeEach(function() {
    xhr = sinon.useFakeXMLHttpRequest();
    requests = [];
    xhr.onCreate = function(req) {
      requests.push(req);
    };
  });

  afterEach(function() {
    xhr.restore();
  });
  // ...
});
```

现在我们可以加入第一个测试，这个测试是与模板加载相关的。它会测试当带有`templateUrl`的指令被编译时，是否生成了一个对指定 URL 的 GET 请求：

```js
it('fetches the template', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html'
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');

    $compile(el);
    $rootScope.$apply();
    
    expect(requests.length).toBe(1);
    expect(requests[0].method).toBe('GET');
    expect(requests[0].url).toBe('/my_directive.html');
  });
});
```

注意，这里`$apply`是在`$compile`的后面调用的。我们需要在`$http`服务里使用 Promise 链。

我们需要在`compileTemplateUrl`中创建 HTTP 请求。在此之前，它需要能访问到指令对象，所以我们会传入这个参数：

_src/compile.js_

```js
if (directive.templateUrl) {
  compileTemplateUrl(directive, $compileNode);
  // return false;
} else if (directive.compile) {
```

我们可以使用`$http`服务的`get`方法去创建一个实际的请求：

```js
function compileTemplateUrl(directive, $compileNode) {
  // $compileNode.empty();
  $http.get(directive.templateUrl);
}
```

我们还没有在`$compile`引入`$http`服务，所以我们需要在`CompileProvider.$get`里面注入这个参数：

```js
this.$get = ['$injector', '$parse', '$controller', '$rootScope', '$http',
    function($injector, $parse, $controller, $rootScope, $http) {
```

现在就达成了通过测试的条件了。

当模板被接收到时需要做些什么呢？显然，我们需要将模板转化、生成为元素内容：

```js
it('populates element with template', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html'
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');

    $compile(el);
    $rootScope.$apply();
    
    requests[0].respond(200, {}, '<div class="from-template"></div>');
    expect(el.fnd('> .from-template').length).toBe(1);
  });
});
```

我们可以通过`$http`调用后返回的 Promise 对象的`success`回调函数来处理。这个回调函数的第一个参数就时相应体，也就是模板 HTML 代码：

_src/compile.js_

```js
function compileTemplateUrl(directive, $compileNode) {
  // $compileNode.empty();
  $http.get(directive.templateUrl).success(function(template) {
    $compileNode.html(template);
  });
}
```

现在我们获取到了处于可以恢复指令编译状态下的 DOM。这意味着当我们接收到了指令模板，我们可以让当前指令的`compile`函数被正常调用：

_test/compile_spec.js_

```js
it('compiles current directive when template received', function() {
  var compileSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html',
        compile: compileSpy
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');

    $compile(el);
    $rootScope.$apply();
    
    requests[0].respond(200, {}, '<div class="from-template"></div>');
    expect(compileSpy).toHaveBeenCalled();
  });
});
```

同样可以继续进行编译过程的还是同一元素上的其他指令，我们在`applyDirectivesToNode`函数中跳过了它们的编译，现在需要继续它们的编译：

```js
it('resumes compilation when template received', function() {
  var otherCompileSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html'
      };
    },
    myOtherDirective: function() {
      return {
        compile: otherCompileSpy
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');

    $compile(el);
    $rootScope.$apply();
    
    requests[0].respond(200, {}, '<div class="from-template"></div>');
    expect(otherCompileSpy).toHaveBeenCalled();
  });
});
```

怎么实现呢？我们需要的是在接收到这个模板时重启`applyDirectivesToNode`的处理逻辑，但只针对还没有进行编译的指令。而这就是我们将要做的事。

首先，`compileTemplateUrl`不仅要可以访问到当前指令，还应该可以访问到所有还没应用上的指令。换句话说，它需要当前所有指令的一个子数组，收集的指令从当前元素的索引开始。此外，我们也会传入当前元素的`Attributes`对象，因为我们之后会用到：

_src/compile.js_

```js
if (directive.templateUrl) {
  compileTemplateUrl(_.drop(directives, i), $compileNode, attrs);
  return false;
```

这个`i`变量还没有，我们应该在遍历指令的`_.forEach`循环函数中加入这个参数，它指向当前指令的索引：

```js
_.forEach(directives, function(directive, i) {
  // ...
});
```

现在，`compileTemplateUrl`的第一个参数将变成一个数组，而使用`templateUrl`的指令将会是数组的第一个元素：

```js
function compileTemplateUrl(directives, $compileNode, attrs) {
  var origAsyncDirective = directives[0];
  // $compileNode.empty();
  $http.get(origAsyncDirective.templateUrl).success(function(template) {
    // $compileNode.html(template);
  });
}
```

现在我们就有了重新调用`applyDirectivesToNode`的所有基础了：

```js
function compileTemplateUrl(directives, $compileNode, attrs) {
  // var origAsyncDirective = directives[0];
  // $compileNode.empty();
  // $http.get(origAsyncDirective.templateUrl).success(function(template) {
  //   $compileNode.html(template);
    applyDirectivesToNode(directives, $compileNode, attrs);
  // });
}
```

但这里还有一个问题，我们现在遇到`templateUrl`属性的指令就会重启编译。这意味着`applyDirectivesToNode`会马上遇上`templateUrl`并再次停止编译。然后我们就永远陷在不断获取模板的循环里。

我们可以通过先从 directives 数组中移除第一个指令（带异步模板的指令）来解决：

```js
function compileTemplateUrl(directives, $compileNode, attrs) {
  var origAsyncDirective = directives.shift();
  // $compileNode.empty();
  // $http.get(origAsyncDirective.templateUrl).success(function(template) {
  //   $compileNode.html(template);
  //   applyDirectivesToNode(directives, $compileNode, attrs);
  // });
}
```

然后我们就用一个新的指令对象来补充 directives 数组，这个新指令对象的属性都跟第一个指令的属性一样，除了`templateUrl`被设置为`null`：

```js
function compileTemplateUrl(directives, $compileNode, attrs) {
  // var origAsyncDirective = directives.shift();
  var derivedSyncDirective = _.extend({},
    origAsyncDirective, {
      templateUrl: null
    }
  );
  // $compileNode.empty();
  // $http.get(origAsyncDirective.templateUrl).success(function(template) {
    directives.unshift(derivedSyncDirective);
  //   $compileNode.html(template);
  //   applyDirectivesToNode(directives, $compileNode, attrs);
  // });
}
```

我们在重启编译过程中还漏了一步，就是对子节点的编译。`applyDirectivesToNode`并没有完成这步工作。

_test/compile_spec.js_

```js
it('resumes child compilation after template received', function() {
  var otherCompileSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html'
      };
    },
    myOtherDirective: function() {
      return {
        compile: otherCompileSpy
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');

    $compile(el);
    $rootScope.$apply();
    
    requests[0].respond(200, {}, '<div my-other-directive></div>');
    expect(otherCompileSpy).toHaveBeenCalled();
  });
});
```

我们需要做的仅仅是对当前节点的子节点调用`compileNodes`，这时来自模板的子节点都已经包含在里面了：

```js
function compileTemplateUrl(directives, $compileNode, attrs) {
  // var origAsyncDirective = directives.shift();
  // var derivedSyncDirective = _.extend({}, origAsyncDirective, {
  //   templateUrl: null
  // });
  // $compileNode.empty();
  // $http.get(origAsyncDirective.templateUrl).success(function(template) {
  //   directives.unshift(derivedSyncDirective);
  //   $compileNode.html(template);
  //   applyDirectivesToNode(directives, $compileNode, attrs);
    compileNodes($compileNode[0].childNodes);
  // });
}
```