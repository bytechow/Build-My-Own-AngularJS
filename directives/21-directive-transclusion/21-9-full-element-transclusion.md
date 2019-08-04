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