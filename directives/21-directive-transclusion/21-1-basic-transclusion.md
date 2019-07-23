### 简单的 transclusion（Basic Transclusion）

最简单的 transclusion 用例就是：当元素上使用了一个 transclusion 指令，元素里面的子节点会被迁移到指令系统的指定位置上去。

![compile and link](/assets/basic-transclusion.png)

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