### 完整元素的 transclusion（Full Element Transclusion）

本章的剩余部分我们会关注一种跟之前构建的 transclusion 特性稍微不同的用例。这也需要我们对 transclusion 特性进行一些扩展。

通常我们学习 transclusion 时，我们是在介绍如何把一个元素的内容转移另一个模板的一个元素内。这也是本章我们已经介绍了的内容，这也是大家平时讨论 transclusion 时脑袋里会想起的事情。

其实还有另一种类型的“transclusion”，它通过`transclude`配置选项来实现，被称为 full element transclusion。`transclude`属性值将不会被设置为`true`，而是`element`。

表面上，这跟之前我们说的那种普通的 transclusion 没什么差别：full element transclusion 会把整个元素加上 transclusion 指令本身都作为 transclusion 内容，而普通的 transclusion 只会把它的子节点作为 transclusion 内容。

而且，这还不是它们的唯一区别。事实上`transclude: ‘element’`已经被设计成用于一种与普通  transclusion 完全不同的用例。full element transclusion 只是碰巧使用了同一个配置项，并且两种 transclusion 共用了不少的实现代码。

使用场景不同在于`transclude: true`是用于转移模板的一部分到另一个模板。而`transclude: 'element'`是不移动元素的位置，但对元素内容的处理提供了更加控制：比如这个元素可以在满足一些条件的情况才被加入到其他模板，比如可以延迟插入到其他模板的时间，比如它也可以被多次插入。

其实`ngIf`与`ngRepeat`这些指令都是以这种 transclusion 为基石的，而 full element transclusion 本身也是为了要支持这些指令而诞生的。我们开发这种特性，主要还是基于之前实现的克隆和作用域管理这两个特性。

下面我就来看看这个 full element transclusion 究竟是什么样的。建立第一个对它的单元测试。当你在指令上使用`transclude: 'element'`时，带有这个指令的那个元素实际上会在编译阶段消失不见：

_test/compile_spec.js_

```js
describe('element transclusion', function() {

  it('removes the element from the DOM', function() {
    var injector = makeInjectorWithDirectives({
      myTranscluder: function() {
        return {
          transclude: 'element'
        };
      }
    });
    injector.invoke(function($compile) {
      var el = $('<div><div my-transcluder></div></div>');
  
      $compile(el);
  
      expect(el.is(':empty')).toBe(true);
    });
  });
  
});
```

作为开始，这个测试用例有点奇怪，但它确实也是我们希望出现的结果。

我们会在`applyDirectivesToNode`种加入 element transclusion，在这里我们可以通过`if-else`条件语句来区分它和普通 transclusion：

_src/compile.js_

```js
if (directive.transclude) {
  // if (hasTranscludeDirective) {
  //   throw 'Multiple directives asking for transclude';
  // }
  // hasTranscludeDirective = true;
  if (directive.transclude === 'element') {
    $compileNode.remove();
  } else {
    // var $transcludedNodes = $compileNode.clone().contents();
    // childTranscludeFn = compile($transcludedNodes);
    // $compileNode.empty();
  }
}
```

并不只是消失了一个元素这么简单，我们会在它原有的位置加入一行 HTML 注释。这个注释里面有指令名称，然后时一个冒号和两个空格键字符：

_test/compile_spec.js_

```js
it('replaces the element with a comment', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: 'element'
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div><div my-transcluder></div></div>');

    $compile(el);
    
    expect(el.html()).toEqual('<!-- myTranscluder: -->');
  });
});
```

注意，这里我们在外层包裹了一层`<div>`，这样我们可以更容易地发现变化。

我们可以使用 DOM 标准的`document.createElement()`函数来创建这个注释。我们可以使用 jQuery/jqLite 的`replaceWith()`函数来把当前节点替换为这个新创建的注释节点。这个函数会负责操作 DOM 以让注释替代原有的内容。

注意，这里并不会替换`$compileNode`变量本身的内部内容，虽然这个 API 像是在做这个操作。`$compileNode`会内部存在的依然是原有的元素。

_src/compile.js_

```js
if (directive.transclude) {
  // if (hasTranscludeDirective) {
  //   throw 'Multiple directives asking for transclude';
  // }
  // hasTranscludeDirective = true;
  // if (directive.transclude === 'element') {
    $compileNode.replaceWith(
      $(document.createComment(' ' + directive.name + ': ')));
  // } else {
  //   var $transcludedNodes = $compileNode.clone().contents();
  //   childTranscludeFn = compile($transcludedNodes);
  //   $compileNode.empty();
  // }
}
```

如果在原有元素的指令属性有值，我们就需要把这个值额外加入到生成的注释节点。这样的话，我们生成的注释节点看上去就像是使用了注释式指令一样：

_test/compile_spec.js_

```js
it('includes directive attribute value in comment', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {transclude: 'element'};
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div><div my-transcluder=42></div></div>');

    $compile(el);
    
    expect(el.html()).toEqual('<!-- myTranscluder: 42 -->');
  });
});
```

我们可以直接从当前的 Attributes 对象中把对应的属性值拿出来用，这样就能通过测试了：

_src/compile.js_

```js
if (directive.transclude === 'element') {
  // $compileNode.replaceWith($(document.createComment(
    ' ' + directive.name + ': ' + attrs[directive.name] + ' '
  // )));
} else {
  // ...
}
```

如果我们将 DOM 里面的元素节点替换为注释节点，那我们到底应该传递什么给指令的编译和链接函数呢？实际上我们传递的是注释节点。所以当我们遇到一个定义了`transclude: 'element'`的指令，它的编译和链接函数接收到的都是一个注释节点，这确实够奇怪的了：

_test/compile_spec.js_

```js
it('calls directive compile and link with comment', function() {
  var gotCompiledEl, gotLinkedEl;
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: 'element',
        compile: function(compiledEl) {
          gotCompiledEl = compiledEl;
          return function(scope, linkedEl) {
            gotLinkedEl = linkedEl;
          };
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div><div my-transcluder></div></div>');

    $compile(el)($rootScope);
    
    expect(gotCompiledEl[0].nodeType).toBe(Node.COMMENT_NODE);
    expect(gotLinkedEl[0].nodeType).toBe(Node.COMMENT_NODE);
  });
});
```

这里的关键是我们会将`$compileNode`变量替换为一个持有新创建的注释节点的变量，因为这才是我们要传递到编译和链接函数里去的变量。所以，我们需要对`$compileNode`重新赋值，但在此之前，我们需要将它的原始值（原始的元素节点）存放到另一个变量：

_src/compile.js_

```js
if (directive.transclude === 'element') {
  var $originalCompileNode = $compileNode;
  $compileNode = $(document.createComment(
    ' ' + directive.name + ': ' + attrs[directive.name] + ' '
  ));
  $originalCompileNode.replaceWith($compileNode);
} else {
  // ...
}
```

所以目前`$originalCompileNode`会保存当前指令所在的元素，而`$compileNode`则会保存新创建的注释节点。原始元素节点不会被附加到其他地方，而替代的注释节点将会出现在本来属于元素节点的位置。

有趣的是，如果在同一个元素上有低优先级指令，它们也是需要被编译的，即使我们会从 DOM 中移除这个元素：

_test/compile_spec.js_

```js
it('calls lower priority compile with original', function() {
  var gotCompiledEl;
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        priority: 2,
        transclude: 'element'
      };
    },
    myOtherDirective: function() {
      return {
        priority: 1,
        compile: function(compiledEl) {
          gotCompiledEl = compiledEl;
        }
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div><div my-transcluder my-other-directive></div></div>');

    $compile(el);
    
    expect(gotCompiledEl[0].nodeType).toBe(Node.ELEMENT_NODE);
  });
});
```

当这些指令确实在当前代码实现的条件下进行了编译，它是跟注释节点一起出现的，但有点令人感觉奇怪的是，注释节点上并没有这些指令。上面这个测试会失败，因为它希望编译发生在元素节点上，而不是注释节点。

除此之外，子节点上的指令应该也会被编译：

```js
it('calls compile on child element directives', function() {
  var compileSpy = jasmine.createSpy();
  var injector = makeInjectorWithDirectives({
    myTranscluder: function() {
      return {
        transclude: 'element'
      };
    },
    myOtherDirective: function() {
      return {
        compile: compileSpy
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $(
      '<div><div my-transcluder><div my-other-directive></div></div></div>');

    $compile(el);
    
    expect(compileSpy).toHaveBeenCalled();
  });
});
```

我们怎么来满足这些需求呢？我们要做的是把编译过程分割成两部分：我们需要在当前指令上停止编译，让它变成当前编译的最后一个指令。然后我们再在刚才被替换了的元素上启动另一个编译。

要停止当前的编译过程，我们可以使用之前开发的 terminal priority，将当前指令的 priority 赋值给 terminal priority：

_src/compile.js_

```js
if (directive.transclude === 'element') {
  // var $originalCompileNode = $compileNode;
  // $compileNode = $(document.createComment(
  //   ' ' + directive.name + ': ' + attrs[directive.name] + ' '
  // ));
  // $originalCompileNode.replaceWith($compileNode);
  terminalPriority = directive.priority;
} else {
  // ...
}
```

现在当前元素上的低优先级指令就不会进行编译了，而元素子节点也同样不会进行编译。虽然我们刚才写的最近的一个单元测试依然失败了，但失败原因跟之前不同了。