### 编译Element指令的DOM（Compiling The DOM with Element Directives）

现在我们可以注册指令了，接下来可以使用它们了。使用的过程称为“DOM 编译”，这也是`$compile`的主要功能。

现在我们假设有一个名为`myDirective`的指令。我们可以把这个指令编写为一个会返回对象的函数：

```js
myModule.directive('myDirective', function() {
  return {
  };
});
```

返回的对象就是`指令定义对象`。它的属性将作为配置指令行为的依据。其中有一个属性是`compile`。有了这个属性，我们就可以定义指令的编译函数。这是一个会在`$compile`遍历 DOM 元素时会调用的函数。它会接收一个参数，这个参数就是要应用指令的那个 DOM 元素：

```js
myModule.directive('myDirective', function() {
  return {
    compile: function(element) {
  
    }
  };
});
```

当我们有了这样一个指令，我们就可以使用一个匹配的元素名来应用该指令：

```js
<my-directive></my-directive>
```

让我们把上面的例子合并起来作为一个单元测试。在单元测试里，我们需要创建一个注射器，这个注射器包含一个指令。本部分我们会多次使用到这样的一个注射器，所以我们会把这个过程封装成一个帮助函数：

_test/compile_spec.js_

```js
function makeInjectorWithDirectives() {
  var args = arguments;
  return createInjector(['ng', function($compileProvider) {
    $compileProvider.directive.apply($compileProvider, args);
  }]);
}
```

这个帮助函数会创建一个由两个模块组成的注射器：ng 模块和一个函数式模块，在函数式模块中将会利用注入的`$compileProvider`注册一个指令。

我们可以直接在新的单元测试中应用这个函数：

_test/compile_spec.js_

```
it('compiles element directives from a single element', function() {
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', true);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<my-directive></my-directive>');
    $compile(el);
    expect(el.data('hasCompiled')).toBe(true);
  });
});
```

这个测试将会做以下几件事情：

1. 会创建一个带有`myDirective`指令的模块，并会创建一个注射器。
2. 会利用 jQuery 对`<my-directive>`指令进行解析，解析成为 jQuery DOM 片段。
3. 会注入`$compile`函数，并使用第2步生成的 jQuery 对象作为参数调用该函数。

在指令编译函数内部，我们就只需要对元素添加一个 data 属性，这让我们可以在单元测试的最后检测指令是否真的被应用上了。

我们现在需要在这个测试文件中加入 jQuery：

_test/compile_spec.js_

```js
'use strict';

var _ = require('lodash');
var $ = require('jquery');
var publishExternalAPI = require('../src/angular_public');
var createInjector = require('../src/injector');
```

> 正如本书之前介绍的，我们会使用 jQuery 来提供低层次的 DOM 检查和操作能力。真正的 AngularJS 本身并没有太多地依赖 jQuery，而是会集成一个 jQuery 的子集，称为`jqLite`。由于 DOM 原理在本书不是重点，我们会直接使用 jQuery。也就是说，当我们处理指令时，AngularJS 会返回的是 jqLite 对象，而本书返回的是 jQuery 对象。

传入到`$compile` 函数的参数决不限于单个 DOM 元素。这个参数也可能是几个元素的集合。这次我们用两个指令作为兄弟元素进行测试，看看编译函数会不会对它们进行分别调用：

```js
it('compiles element directives found from several elements', function() {
  var idx = 1;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      compile: function(element) {
        element.data('hasCompiled', idx++);
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $('<my-directive></my-directive><my-directive></my-directive>');
    $compile(el);
    expect(el.eq(0).data('hasCompiled')).toBe(1);
    expect(el.eq(1).data('hasCompiled')).toBe(2);
  });
});
```

要让这个测试通过，我们需要在代码新增一些东西。下面我们会一个个过，最后我们会展示更新过后的`CompileProvider`的全部源代码。

首先，`CompileProvider`里的`$get`方法需要返回一个值，这个值就是我们刚才在单元测试调用的`$compile`函数：

_src/compile.js_

```js
this.$get = function(){

  function compile($compileNodes){

  }

  return compile;
}
```

正如我们在单元测试中看到的，这个函数将会接受需要编译的 DOM 节点作为它参数。

> `$compile`的美元符号前缀经常用在区分 jQuery 封装的 DOM 节点和原生的 DOM 节点。这已经成为 jQuery 界的传统。不幸的是，AngularJS 也用这种方式来标记框架内提供的组件，所以这时候很难区分到底是出于哪种目的使用美元符号前缀。上面的例子中，我们用`$compileNodes`代表一个或多个相关的 jQuery 封装的 DOM 节点。

我们在`compile`里会做的是调用另一个局部函数`compileNodes`。目前看来，这种调用方式有点像是在绕弯子，但我们在之后的开发中，会发现我们确实需要这种区分方式。

```js
this.$get = function() {

  function compile($compileNodes) {
    return compileNodes($compileNodes);
  }

  function compileNodes($compileNodes) {

  }
  
  return compile;
};
```

在`compileNodes`中，我们会遍历传入的每一个 jQuery 对象，从而实现对每一个节点进行独立处理。对于每一个节点，我们会查找任何能跟该节点匹配上的节点，这个查找过程我们会用一个新函数`collectDirectives`实现：

```js

```