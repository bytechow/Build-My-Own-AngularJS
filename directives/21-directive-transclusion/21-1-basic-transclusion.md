### 简单的 transclusion（Basic Transclusion）

最简单的 transclusion 用例就是：当元素上使用了一个 transclusion 指令，元素里面的子节点会被迁移到指令系统的指定位置上去。

![compile and link](/assets/21-direction-transclusion/basic-transclusion.png)

当一个指令定义对象包含了`transclude: true`这个特性。这个功能第一个的可见影响是当前元素的子节点会从原来的 DOM 中消失：

_test/compile_spec.js_

```js
describe('transclude', function() {

  it('removes the children of the element from the DOM', function() {
    var injector = makeInjectorWithDirectives({
      myTranscluder: function() {
        return {
          transclude: true
        };
      }
    });
    injector.invoke(function($compile) {
      var el = $('<div my-transcluder><div>Must go</div></div>');
      
      $compile(el);
      
      expect(el.is(':empty')).toBe(true);
    });
  });

});
```

这个我们实现起来很简单。当在`applyDirectivesToNode`中编译指令时，我们可以检查看指令是否带有值为`true`的 transclude 属性，如果是就把节点都清空。我们会在遍历指令时做这个检查，顺序介于对`controller`和`template`属性的处理之间：

_src/compile.js_

```js
function applyDirectivesToNode(
  directives, compileNode, attrs, previousCompileContext) {
  // ...
  
  _.forEach(directives, function(directive, i) {
    // ...
    
    if (directive.controller) {
      controllerDirectives = controllerDirectives || {};
      controllerDirectives[directive.name] = directive;
    }
    if (directive.transclude) {
      $compileNode.empty();
    }
    if (directive.template) {
      // ...
    }
    // ..
  });
  // ...
}
```

在配置了 transclusion 的情况下，这里我们只是简单地把节点内容都清空了。你可能也能猜到这不是所有的工作。那我们到底要怎么处理这些子节点呢？

有一件事会发生，就是这些节点都会被编译。目前只是因为我们在`compileNodes`遍历这些节点之前移除了这些节点，所以它们才没有被编译。如果我们假设它们已经被编译的话，下面这个测试会失败：

_test/compile_spec.js_

```js
it('compiles child elements', function() {
  var insideCompileSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true
      };
    },
    insideTranscluder: function() {
      return {
        compile: insideCompileSpy
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div my-transcluder><div inside-transcluder></div></div>');

    $compile(el);
    
    expect(insideCompileSpy).toHaveBeenCalled();
  });
});
```

因此我们的确需要编译子节点，但在从原来进行编译的 DOM 树上移除它们之后，应该继续持有子节点。我们可以通过为每个单独的子节点分别调用`compile`服务来同时实现满足这两个条件。实际上，进行 translusion 的内容都会由一个单独的、独立的编译进程进行编译：

_src/compile.js_

```js
if (directive.transclude) {
  var $transcludedNodes = $compileNode.clone().contents();
  compile($transcludedNodes);
  // $compileNode.empty();
}
```

注意，我们在获取要编译的节点的内容之前是先对其进行克隆。这样做的目的是在我们清空节点内容之前，我们依然持有进行了 translusion 的内容的克隆版本。

目前我们是对进行了 translusion 的节点进行编译，但我们在编译之后依然会丢弃它们。它们没有用于任何地方，跟本页介绍的内容毫无关联。而我们想要的其实是允许把这些元素通过 transclusion 转移到其他地方。但在哪呢，又要怎么实现呢？

要明确这些元素会在哪里进行 transclusion 的这个问题，是我们在这个框架中无法解决的问题。因为这个是由应用开发者自行确定的。我们能做的就让这些元素能被应用开发者使用，这样他们就可以把它们放到自己想放的地方去。

要实现这个功能，我们需要对要进行 translusion 的指令链接函数加入新的、第五个参数。这个函数就是 transclusion 函数。这个函数能让指令的创建者访问到已经进行 translusion 的内容。下面的这个测试用例就是用于测试指令到底有没有进行 translusion：它会使用 translusion 函数来获取已经进行 translusion 的内容，并把内容放到模板之中：

_test/compile_spec.js_

```js
it('makes contents available to directive link function', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true,
        template: '<div in-template></div>',
        link: function(scope, element, attrs, ctrl, transclude) {
          element.fnd('[in-template]').append(transclude());
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-transcluder><div in-transcluder></div></div>');
    $compile(el)($rootScope);
    expect(el.fnd('> [in-template] > [in-transcluder]').length).toBe(1);
  });
});
```

当指令使用了 translusion，就可以使用指令链接函数的第五个参数——translusion 函数。下面我们来介绍如何创建这个函数并传入它。

在`applyDirectivesToNode`中，我们需要引入一个新的跟踪变量（tracking variable），叫做`childTranscludeFn`：

_src/compile.js_

```js
function applyDirectivesToNode(
  directives, compileNode, attrs, previousCompileContext) {
  // previousCompileContext = previousCompileContext || {};
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // var preLinkFns = previousCompileContext.preLinkFns || [];
  // var postLinkFns = previousCompileContext.postLinkFns || [];
  // var controllers = {};
  // var newScopeDirective;
  // var newIsolateScopeDirective = previousCompileContext.newIsolateScopeDirective;
  // var templateDirective = previousCompileContext.templateDirective;
  // var controllerDirectives = previousCompileContext.controllerDirectives;
  var childTranscludeFn;
  
  // ...
}
```

在这个变量中，我们会保存用于编译被 translusion 的内容而调用`compile`的函数的返回值。也就是说，这将会是这些内容的公共链接函数：

_src/compile.js_

```js
if (directive.transclude) {
  // var $transcludedNodes = $compileNode.clone().contents();
  childTranscludeFn = compile($transcludedNodes);
  // $compileNode.empty();
}
```

为了通过测试用例，现在我们能做的最简单的事就是对这个公共链接函数进行修改，以便这个函数能够返回它链接过的节点：

```js
return function publicLinkFn(scope) {
  $compileNodes.data('$scope', scope);
  compositeLinkFn(scope, $compileNodes);
  return $compileNodes;
};
```

我们这样做的目的就是让属于被 translusion 的内容的公共链接函数能过作为 translusion 函数，传递给指令：

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  
  // ...
  
  _.forEach(preLinkFns, function(linkFn) {
    linkFn(
      // linkFn.isolateScope ? isolateScope : scope,
      // $element,
      // attrs,
      // linkFn.require && getControllers(linkFn.require, $element),
      childTranscludeFn
    );
  });
  // if (childLinkFn) {
  //   var scopeToChild = scope;
  //   if (newIsolateScopeDirective && newIsolateScopeDirective.template) {
  //     scopeToChild = isolateScope;
  //   }
  //   childLinkFn(scopeToChild, linkNode.childNodes);
  // }
  _.forEachRight(postLinkFns, function(linkFn) {
    linkFn(
      // linkFn.isolateScope ? isolateScope : scope,
      // $element,
      // attrs,
      // linkFn.require && getControllers(linkFn.require, $element),
      childTranscludeFn
    );
  });
}
```

> 注意，这个 translusion 函数将会传递到节点上的所有指令里面去，包括含有`transclude: true`属性和不含该属性的指令。

我们现在接触到了一个关键点：本质上，transclusion 函数就是一个链接函数。现在，它就是一个原原本本的、用于进行了 translusion 的内容的公共链接函数。但事情并没有这么简单。比如，我们现在就没有考虑到作用域的管理。但 transclusion 函数实际上就是链接函数的这个基本点不会改变。

在我们进行作用域的管理之前，我们会对 transclusion 的使用添加一个限制：就像指令模板一样，我们在一个元素上只能使用一次 transclusion。在（同一个元素上的）两个指令都使用 transclusion 并没有多大意义，因为第一次 transclusion 已经把节点里原本的内容都清空掉了，第二个 transclusion 指令也就没法操作了。因此当（同一个元素上）使用了超过两个 transclusion 指令，我们就会跑出一个明确的错误：

_test/compile_spec.js_

```js
it('is only allowed once per element', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: true
      };
    },
    mySecondTranscluder: function() {
      return {
        transclude: true
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div my-transcluder my-second-transcluder></div>');
    expect(function() {
      $compile(el);
    }).toThrow();
  });
});
```

要跟踪检查这种情况，我们需要在`applyDirectivesToNode`中使用一个新变量，这个变量就是一个标识，记录之前是否在当前元素上出现过 transclusion 指令：

_src/compile.js_

```js
function applyDirectivesToNode(
  directives, compileNode, attrs, previousCompileContext) {
  previousCompileContext = previousCompileContext || {};
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // var preLinkFns = previousCompileContext.preLinkFns || [];
  // var postLinkFns = previousCompileContext.postLinkFns || [];
  // var controllers = {};
  // var newScopeDirective;
  // var newIsolateScopeDirective = previousCompileContext.newIsolateScopeDirective;
  // var templateDirective = previousCompileContext.templateDirective;
  // var controllerDirectives = previousCompileContext.controllerDirectives;
  var childTranscludeFn, hasTranscludeDirective;
  // ...
}
````

在 transclusion 指令出现时，我们就会对这个标识进行检查：

```js
if (directive.transclude) {
  if (hasTranscludeDirective) {
    throw 'Multiple directives asking for transclude';
  }
  hasTranscludeDirective = true;
  // var $transcludedNodes = $compileNode.clone().contents();
  // childTranscludeFn = compile($transcludedNodes);
  // $compileNode.empty();
}
```