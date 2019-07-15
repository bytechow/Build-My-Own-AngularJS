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

另一个要注意的是，这时当前元素的内容应该被清空了。元素内容会被即将载入的模板内容代替，但我们需要立即清除所有的旧内容，这样旧内容就不会被不必要地编译了：

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