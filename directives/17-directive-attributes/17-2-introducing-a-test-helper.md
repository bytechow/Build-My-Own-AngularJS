### 引入一个测试帮助函数（Introducing A Test Helper）

在继续学习指令属性的下一部份内容之前，我们可以做一些事情来减少单元测试的重复代码。目前的单元测试有以下共同点：

1. 注册一个指令
2. 编译一个 DOM 片段
3. 获取属性对象
4. 对编译结果进做一些检查

对此，我们可以在指令属性对应的`describe`代码块中新增一个帮助函数：

_test/compile_spec.js_

```js
function registerAndCompile(dirName, domString, callback) {
  var givenAttrs;
  var injector = makeInjectorWithDirectives(dirName, function() {
    return {
      restrict: 'EACM',
      compile: function(element, attrs) {
        givenAttrs = attrs;
      }
    };
  });
  injector.invoke(function($compile) {
    var el = $(domString);
    $compile(el);
    callback(el, givenAttrs);
  });
}
```

这个函数会接收三个参数：要注册的指令名称、要进行解析和编译的 DOM 字符串和在最后要做检查的回调函数。这个回调函数会接收元素和属性对象作为参数：

```js
it('passes the element attributes to the compile function', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive my-attr="1" my-other-attr="two"></my-directive>',
    function(element, attrs) {
      expect(attrs.myAttr).toEqual('1');
      expect(attrs.myOtherAttr).toEqual('two');
    }
  );
});
it('trims attribute values', function() {
  registerAndCompile(
    'myDirective',
    '<my-directive my-attr=" val "></my-directive>',
    function(element, attrs) {
      expect(attrs.myAttr).toEqual('val');
    }
  );
});
```