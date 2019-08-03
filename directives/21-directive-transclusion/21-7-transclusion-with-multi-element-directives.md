### 跨元素指令的 transclusion（Transclusion with Multi-Element Directives）

第二种需要特别注意的 transclusion 就是与跨元素指令结合的 transclusion。你可以会说这两个特性用在一起本来就没什么意义：因为跨元素指令是以不同的两个兄弟节点作为起点和终点的，那到底用哪个节点的子元素作为 transclude 内容的存放位置呢？

答案并不是显而易见的，而碰巧，Angular 也没有为此做什么特殊逻辑处理——它只是会像 jQuery 或 jqLite 的 DOM 操作函数做的一样对待跨元素指令。然而，Angular 确实有对这种情况有基本的支持。我们需要在`compile_spec.js`的`describe('transclude')`测试模块中加入下面这个测试用例：

_test/compile\_spec.js_

```js
it('can be used with multi-element directives', function() {
  var injector = makeInjectorWithDirectives({
    myTranscluder: function($compile) {
      return {
        transclude: true,
        multiElement: true,
        template: '<div in-template></div>',
        link: function(scope, element, attrs, ctrl, transclude) {
          element.find('[in-template]').append(transclude());
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $(
      '<div><div my-transcluder-start><div in-transclude></div></div>' +
      '<div my-transcluder-end></div></div>'
    );
    $compile(el)($rootScope);
    expect(el.find('[my-transcluder-start] [in-template] [in-transclude]').length)
      .toBe(1);
  });
});
```

要通过测试，我们需要做的仅仅是保证用于传递一组元素的、包裹着链接函数的函数，能把 transclusion 函数也传递进去：

_src/compile.js_

```js
function groupElementsLinkFnWrapper(linkFn, attrStart, attrEnd) {
  return function(scope, element, attrs, ctrl, transclude) {
    var group = groupScan(element[0], attrStart, attrEnd);
    return linkFn(scope, group, attrs, ctrl, transclude);
  };
}
```



