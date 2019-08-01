### 带模板 URL 的 transclusion（Transclusion with Tempalte URLs）

在上一章，我们花费了很大的力气建立一套可以暂停和重启的机制来实现异步模板的加载。那这个机制能兼容有 transclusion 的情况吗？

目前的答案会是不能。当使用了`templateUrls`是，节点链接函数目前接收到的绑定 transclusion 函数并不被这套机制里的_延迟的节点链接函数_所支持。但是否使用 transclusion 不应该影响使用`templateUrls`。

让我们先考虑一下使用了`templateUrl`但在链接开始之前仍然接收到模板的情况。让我们在`compile_spec.js`的“templateUrl”部分加入下面这个测试用例：

_test/compile_spec.js_

```js
describe('with transclusion', function() {

  it('works when template arrives frst', function() {
    var injector = makeInjectorWithDirectives({
      myTranscluder: function() {
        return {
          transclude: true,
          templateUrl: 'my_template.html',
          link: function(scope, element, attrs, ctrl, transclude) {
            element.find('[in-template]').append(transclude());
          }
        };
      }
    });
    injector.invoke(function($compile, $rootScope) {
      var el = $('<div my-transcluder><div in-transclude></div></div>');

      var linkFunction = $compile(el);
      $rootScope.$apply();
      requests[0].respond(200, {}, '<div in-template></div>'); // respond frst
      linkFunction($rootScope); // then link
      
      expect(el.find('> [in-template] > [in-transclude]').length).toBe(1);
    });
  });
  
});
```

正如所料，这个单元测试失败了。我们需要做的是，当传递给让_延迟的节点链接函数_一个绑定的 transclusion 函数时，它可以接收并处理。如果模板已经获取到了，而`linkQueue`不再存在，我们可以把它（绑定的 transclusion 函数）传递给普通的节点链接函数：

_src/compile.js_

```js
return function delayedNodeLinkFn(
  _ignoreChildLinkFn, scope, linkNode, boundTranscludeFn) {
  if (linkQueue) {
    linkQueue.push({
      scope: scope,
      linkNode: linkNode
    });
  } else {
    afterTemplateNodeLinkFn(
      afterTemplateChildLinkFn, scope, linkNode, boundTranscludeFn);
  }
};
```