### 文本节点的 interpolation（Text Node Interpolation）

由于我们已经实现了一个能派上用场的 interpolation 服务，我们可以开始介绍它是如何与 Angular 框架中的其余部分进行整合的。

interpolation 的一个最重要的使用场景是在 DOM 中嵌入表达式，包括宿主 HTML 页面和模版。自然我们就想到了在`$compile`服务中进行这些 interpolation，因为`$compile`本来为了找到指令就需要对 DOM 进行遍历。这就意味着我们需要在`compile.js`文件中加入对 interpolation 的支持。

interpolation 表达式在 HTML 中能用在两处地方。你可以把它作为 HTML 的文本嵌入：

```xml
<div>Your username is {{user.username}}</div>
```

这被称为 text node interpolation，因为这种 interpolation 会出现在 DOM 的文本节点上，而不是在元素节点或注释节点上。

我们其实也可以在元素属性上使用 interpolation。

```xml
<img src="picture.png" alt="{{altText}}">
```

不出所料，这种 interpolation 被称为 attribute interpolation。

由于 text node interpolation 实现起来更为简单，我们会先介绍它。

当在文本节点上出现了 interpolation，它需要被替换为表达式的结果值，这个结果值是在文本节点所在的 Scope 上下文语境下求得的。另外，由于我们对表达式进行了脏值检测，所以当它的值发生变化时，这个文本节点也需要被更新。下面是第一个测试用例（放在了`compile_test.js`文件中）：

_src/compile.js_

```js
describe('interpolation', function() {

  it('is done for text nodes', function() {
    var injector = makeInjectorWithDirectives({});
    injector.invoke(function($compile, $rootScope) {
      var el = $('<div>My expression: {{myExpr}}</div>');
      $compile(el)($rootScope);

      $rootScope.$apply();
      expect(el.html()).toEqual('My expression: ');

      $rootScope.myExpr = 'Hello';
      $rootScope.$apply();
      expect(el.html()).toEqual('My expression: Hello');
    });
  });

});
```

要实现这个功能，当在编译期间遇到一个文本节点，我们就会动态生成一个指令，并把它加入到文本节点。我们可以利用生成出来的指令来执行 interpolation 的逻辑。

之前我们并没有对文本节点进行过任何处理，因为`collectDirectives`函数只对`Node.ELEMENT_NODE`和`Node.COMMENT_NODE`这两种节点进行处理。也就是说目前我们没有对除此以外的节点类型应用过指令。现在，这种情况要发生改变了，我们会对这个函数增加处理文本节点的逻辑分支：

```js
function collectDirectives(node, attrs, maxPriority) {
  // var directives = [];
  // var match;
  // if (node.nodeType === Node.ELEMENT_NODE) {
  //   // ...
  // } else if (node.nodeType === Node.COMMENT_NODE) {
  //   // ...
  } else if (node.nodeType === Node.TEXT_NODE) {

  // }
  // directives.sort(byPriority);
  // return directives;
}
```

在这个处理分支中，我们将会调用一个新函数`addTextInterpolateDirective`，这个函数会完成生成并添加指令到节点的工作。它会接受指令集合和节点的文本值作为参数：

```js
} else if (node.nodeType === Node.TEXT_NODE) {
  addTextInterpolateDirective(directives, node.nodeValue);
}
```

这个新函数（你可以把它放在`addDirective`函数下面）的第一个任务就是调用`$interpolate`服务来生成一个 interpolation 函数。我们只需要把节点的文本值传给 interpolation 函数作为参数即可。同时，我们要启用`mustHaveExpressions`标识，这样如果没有东西需要进行 interpolate，我们就不会返回函数了：

```js
function addTextInterpolateDirective(directives, text) {
  var interpolateFn = $interpolate(text, true);
}
```

这需要我们对`$compile`加入一个新依赖——`$interpolate`服务：

```js
this.$get = ['$injector', '$parse', '$controller', '$rootScope','$http', '$interpolate',
function($injector, $parse, $controller, $rootScope, $http, $interpolate) {
```

如果`$interpolate`返回的是一个 interpolation 函数，按照前面说的，我们就需要生成一个指令，并把它加入到`directives`集合中：

```js
function addTextInterpolateDirective(directives, text) {
  var interpolateFn = $interpolate(text, true);
  if (interpolateFn) {
    directives.push({
      priority: 0,
      compile: function() {
        return function link(scope, element) {};
      }
    });
  }
}
```

由于我们是在内部生成指令，这个指令没有走正常的注册流程，而这个注册流程原本会生成`priority`和`compile`属性的默认值。这意味着我们需要在这里定义它们的默认值。我们无需考虑优先级，因为在文本节点上不会再有其他指令，但为了统一，我们还是会设置一个`priority`：

现在我们加入了这个指令后，它就会在`applyDirectivesToNode`中进行编译，然后在节点链接函数中进行连接，跟其他指令没什么区别。

在链接函数中，我们会开始会基于当前的 scope，对表达式的值进行 watch。方便的是，我们可以直接把 interpolation 函数作为 watch function，因为 interpolation 满足 watch 函数的条件：它会接受 scope 作为参数，并返回一个经过 interpolate 的值。这个值正正就是我们希望进行脏值检测的对象。在 listener 函数中，我们会把文本节点的值替换为 interpolation 的结果值。

```js
function addTextInterpolateDirective(directives, text) {
  // var interpolateFn = $interpolate(text, true);
  // if (interpolateFn) {
  //   directives.push({
  //     priority: 0,
  //     compile: function() {
  //       return function link(scope, element) {
          scope.$watch(interpolateFn, function(newValue) {
            element[0].nodeValue = newValue;
          });
  //       };
  //     }
  //   });
  // }
}
```

这样，我们就满足了让文本节点支持像`{{expressions}}`这种表达式的所有条件。

Angular 还会在这个过程中对文本节点进行一些别的操作。它们的出现主要是为了在开发过程中提供帮助，还能支持一些调试工具如[Batarang](https://github.com/angular/batarang)，因为它可以通过 DOM 提供一些关于 interpolation 的信息。举一个例子，如果文本节点内出现了 interpolation，我们就会对其：

_test/compile\_spec.js_

```js
it('adds binding class to text node parents', function() {
  var injector = makeInjectorWithDirectives({});
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div>My expression: {{myExpr}}</div>');
    $compile(el)($rootScope);

    expect(el.hasClass('ng-binding')).toBe(true);
  });
});
```

使用我们对文本节点生成的指令，就可以很方便地在链接阶段对当前元素的父节点添加这个 class：

_src/compile.js_

```js
return function link(scope, element) {
  element.parent().addClass('ng-binding');
  // scope.$watch(interpolateFn, function(newValue) {
  //   element[0].nodeValue = newValue;
  // }); 
};
```



