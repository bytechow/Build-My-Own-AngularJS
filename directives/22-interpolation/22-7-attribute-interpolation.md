### 属性 interpolation（Attribute Interpolation）

第二种 interpolation 是在 DOM 属性上使用的。尽管它基本上跟文本节点上的 interpolation 十分类似，但由于属性可能会用于在元素的不同指令之间进行交互，这会比第一种复杂一点。我们需要保证开发这个功能后不影响指令之间的交互。

属性 interpolation 的基本行为跟文本节点 interpolation 是完全一样的。属性中的 interpolation 会在链接期间被替换，并会持续对变化进行检测：

_test/compile_spec.js_

```js
it('is done for attributes', function() {
  var injector = makeInjectorWithDirectives({});
  injector.invoke(function($compile, $rootScope) {
    var el = $('<img alt="{{myAltText}}">');
    $compile(el)($rootScope);

    $rootScope.$apply();
    expect(el.attr('alt')).toEqual('');
    
    $rootScope.myAltText = 'My favourite photo';
    $rootScope.$apply();
    expect(el.attr('alt')).toEqual('My favourite photo');
  });
});
```