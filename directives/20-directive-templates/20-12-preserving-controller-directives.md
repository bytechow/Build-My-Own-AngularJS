### 保存带控制器的指令（Preserving Controller Directives）

最后，我们需要再次使用这个方法来处理带控制器的指令的映射对象。现在，我们会忽略之前看到的、迁移到延迟的链接过程的控制器配置：

_test/compile_spec.js_

```js
it('sets up controllers for all controller directives', function() {
  var myDirectiveControllerInstantiated, myOtherDirectiveControllerInstantiated;
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        controller: function MyDirectiveController() {
          myDirectiveControllerInstantiated = true;
        }
      };
    },
    myOtherDirective: function() {
      return {
        templateUrl: '/my_other_directive.html',
        controller: function MyOtherDirectiveController() {
          myOtherDirectiveControllerInstantiated = true;
        }
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');

    $compile(el)($rootScope);
    $rootScope.$apply();
    
    requests[0].respond(200, {}, '<div></div>');
    
    expect(myDirectiveControllerInstantiated).toBe(true);
    expect(myOtherDirectiveControllerInstantiated).toBe(true);
  });
});
```

我们应该把`controllerDirectives`放到`previousCompileContext`：

_src/compile.js_

```js
nodeLinkFn = compileTemplateUrl(
  _.drop(directives, i),
  $compileNode,
  attrs, {
    templateDirective: templateDirective,
    newIsolateScopeDirective: newIsolateScopeDirective,
    controllerDirectives: controllerDirectives,
    preLinkFns: preLinkFns,
    postLinkFns: postLinkFns
  }
);
```

然后我们需要在`applyDirectivesToNode`中把带控制器的指令对象拿出来：

```js
function applyDirectivesToNode(
  directives, compileNode, attrs, previousCompileContext) {
  previousCompileContext = previousCompileContext || {};
  var $compileNode = $(compileNode);
  var terminalPriority = -Number.MAX_VALUE;
  var terminal = false;
  var preLinkFns = previousCompileContext.preLinkFns || [];
  var postLinkFns = previousCompileContext.postLinkFns || [];
  var controllers = {};
  var newScopeDirective;
  var newIsolateScopeDirective = previousCompileContext.newIsolateScopeDirective;
  var templateDirective = previousCompileContext.templateDirective;
  var controllerDirectives = previousCompileContext.controllerDirectives;
```