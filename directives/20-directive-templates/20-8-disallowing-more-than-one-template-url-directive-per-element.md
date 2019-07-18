### 不允许元素拥有超过一个带模板URL的指令（Disallowing More Than One Template URL Directive Per
Element）

本章的前面部分我们增加了一个测试，这个测试用于保证一个元素无法使用超过一个带模板的指令。我们应该对其进行扩展，让它也涵盖指令是带有异步模板的情况。当元素上已经发现有带`template`的指令，就不允许再出现带`templateUrl`的指令了：

_test/compile_spec.js_

```js
it('does not allow templateUrl directive after template directive', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        template: '<div></div>'
      };
    },
    myOtherDirective: function() {
      return {
        templateUrl: '/my_other_directive.html'
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div my-directive my-other-directive></div>');
    expect(function() {
      $compile(el);
    }).toThrow();
  });
});
```

我们可以通过在`applyDirectivesToNode`增加对`templateUrl`分支的验证。如果我们之前已经遇到了带模板的指令，我们就抛出异常：

_src/compile.js_

```js
if (directive.templateUrl) {
  if (templateDirective) {
    throw 'Multiple directives asking for template';
  }
  templateDirective = directive;
  // compileTemplateUrl(_.drop(directives, i), $compileNode, attrs);
  // return false;
```

我们对于注册顺序相反的情况也要进行验证。当一个带`templateUrl`的指令已经出现了，我们就不允许元素上再有带`template`的指令了：

_test/compile_spec.js_

```js
it('does not allow template directive after templateUrl directive', function() {
  var injector = makeInjectorWithDirectives({
    myDirective: function() {
      return {
        templateUrl: '/my_directive.html'
      };
    },
    myOtherDirective: function() {
      return {
        template: '<div></div>'
      };
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    $compile(el);
    $rootScope.$apply();
    requests[0].respond(200, {}, '<div class="replacement"></div>');
    expect(el.find('> .replacement').length).toBe(1);
  });
});
```

注意在这个测试中没有出现异常，这是因为检测在异步模板加载完毕时已经结束了，异步模板加载是发生在不同的执行上下文中的。

当我们第二次调用`applyDirectivesToNode`查找第二个指令时，这个检测实际上更棘手。此时，所有的局部变量，包括第一次调用时的`templateDirective`都已经被重置了。

我们需要做的是在两次调用中间，保存我们遇到的带模板指令的信息。实际上，我们需要传递第一次调用`applyDirectivesToNode`时的一些状态给另一个调用，主要通过两次调用中发生的`compileTemplateUrl`来完成。

为了这个目的，我们会引入一个对象，叫做`previousCompileContext`，用于保存我们要保存的状态。它的作用就是在两次调用`applyDirectivesToNode`函数之间传递一些局部变量。目前我们要存放到这个对象的信息仅仅是带模板的指令，但之后会加入更多信息。

这个`previousCompileContext`会作为`compileTemplateUrl`函数的最后一个参数传入：

_src/compile.js_

```js
compileTemplateUrl(
  // _.drop(directives, i),
  // $compileNode,
  // attrs, 
  {templateDirective: templateDirective}
);
```

`compileTemplateUrl`不会对`previousCompileContext`做任何处理，只会在需要二次调用`applyDirectivesToNode`传入作为参数：

```js
function compileTemplateUrl(
  directives, $compileNode, attrs, previousCompileContext) {
  // var origAsyncDirective = directives.shift();
  // var derivedSyncDirective = _.extend({},
  //   origAsyncDirective, {
  //     templateUrl: null
  //   }
  // );
  // var templateUrl = _.isFunction(origAsyncDirective.templateUrl) ?
  //   origAsyncDirective.templateUrl($compileNode, attrs)
  // origAsyncDirective.templateUrl;
  // $compileNode.empty();
  // $http.get(templateUrl).success(function(template) {
  //   directives.unshift(derivedSyncDirective);
  //   $compileNode.html(template);
    applyDirectivesToNode(
      directives, $compileNode, attrs, previousCompileContext);
  //   compileNodes($compileNode[0].childNodes);
  // });
}
```

看回到`applyDirectivesToNode`，我们会接收到这个`previousCompileContext`，并用局部变量`templateDirective`来接收它里面存储的同名属性：

```js
function applyDirectivesToNode(
  directives, compileNode, attrs, previousCompileContext) {
  previousCompileContext = previousCompileContext || {};
  // var $compileNode = $(compileNode);
  // var terminalPriority = -Number.MAX_VALUE;
  // var terminal = false;
  // var preLinkFns = [],
  //   postLinkFns = [],
  //   controllers = {};
  // var newScopeDirective, newIsolateScopeDirective;
  var templateDirective = previousCompileContext.templateDirective;
  // var controllerDirectives;
```

现在我们的单元测试都能通过了。在开发过程中，我们还引入了一个简单的机制用于解决异步模板获取时在两次调用`applyDirectivesToNode`之间保持状态的问题。在本章的剩余部分，我们会对这个对象添加更多状态。