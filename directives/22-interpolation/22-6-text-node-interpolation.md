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

