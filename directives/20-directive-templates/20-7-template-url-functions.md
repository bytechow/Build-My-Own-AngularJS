### 模板 URL 函数（Template URL Functions）

就像行内模板可以用函数而代替字符串进行定义，模板 URL 也可以这样。两个属性的函数签名都一样。也就是有两个参数：当前节点和节点的`Attributes`属性。

_test/compile_spec.js_

```js
it('supports functions as values', function() {
  var templateUrlSpy = jasmine.createSpy()
    .and.returnValue('/my_directive.html');
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: templateUrlSpy
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el);
    $rootScope.$apply();
    expect(requests[0].url).toBe('/my_directive.html');
    expect(templateUrlSpy.calls.frst().args[0][0]).toBe(el[0]);
    expect(templateUrlSpy.calls.frst().args[1].myDirective).toBeDefned();
  });
});
```

它的实现方式也跟之前很类似。我们可以简单地对`templateUrl`进行验证，看它是不是一个函数，如果是的话，我们就调用它：

_src/compile.js_

```js
function compileTemplateUrl(directives, $compileNode, attrs) {
  // var origAsyncDirective = directives.shift();
  // var derivedSyncDirective = _.extend({}, origAsyncDirective, {
  //   templateUrl: null
  // });
  var templateUrl = _.isFunction(origAsyncDirective.templateUrl) ?
    origAsyncDirective.templateUrl($compileNode, attrs) :
    origAsyncDirective.templateUrl;
  // $compileNode.empty();
  $http.get(templateUrl).success(function(template) {
    // directives.unshift(derivedSyncDirective);
    // $compileNode.html(template);
    // applyDirectivesToNode(directives, $compileNode, attrs);
    // compileNodes($compileNode[0].childNodes);
  });
}
```