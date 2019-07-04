### 请求注入控制器（Requiring Controllers）

对于链接包含指令逻辑的函数，控制器是一个便利的替代方法，但这并不是控制器的唯一用处。控制器还能被用于两个不同的指令之间沟通的另一个渠道。这也许是 Angular “跨指令通信”工具中最强大的一种：能注入其它指令的控制器。

一个指令可以“请求注入”其它指令，是通过在指令定义对象中指定指令名为`require`属性的属性值完成的。当设置完成以后，并且引入的指令与当前指令存在于同一个元素上，被引入的指令控制器将会赋值到当前正在注入的指令的链接函数的第四个参数上。

事实上，一个指令可以获取另一个指令控制器的访问权，也就是可以访问里面所有公开的数据和函数了。这就是即使指令使用一个独立作用域也允许注入的情况：

_test/compile_spec.js_

```js
it('can be required from a sibling directive', function() {
  function MyController() {}
  var gotMyController;
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.directive('myDirective', function() {
      return {
        scope: {},
        controller: MyController
      };
    });
    $compileProvider.directive('myOtherDirective', function() {
      return {
        require: 'myDirective',
        link: function(scope, element, attrs, myController) {
          gotMyController = myController;
        }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-other-directive></div>');
    $compile(el)($rootScope);
    expect(gotMyController).toBeDefned();
    expect(gotMyController instanceof MyController).toBe(true);
  });
});
```

这里我们有两个指令，`myDirective`和`myOtherDirective`，都应用在同一个元素上。`myDirective`定义了一个控制器和独立作用域。`myOtherDirective`并未定义控制器和独立作用域，只是引入了`myDirective`。我们会检查这个`myDirective`的控制器是否传入到`myOtherDirective`的链接函数中了。

我们先在指令的链接函数中加入一些关于`require`标识的信息。因为我们在`applyDirectivesToNode`的指令便利中调用了`addLinkFns`，我们就在这里传递指令的`require`属性即可：

_src/compile.js_

```js
if (directive.compile) {
  // var linkFn = directive.compile($compileNode, attrs);
  // var isolateScope = (directive === newIsolateScopeDirective);
  // var attrStart = directive.$$start;
  // var attrEnd = directive.$$end;
  var require = directive.require;
  // if (_.isFunction(linkFn)) {
    addLinkFns(null, linkFn, attrStart, attrEnd, isolateScope, require);
  // } else if (linkFn) {
    addLinkFns(
      linkFn.pre, linkFn.post, attrStart, attrEnd, isolateScope, require);
  // }
}
```

在`addLinkFns`函数中，我们会使用这个参数来传递`require`属性给链接前（pre-link）和链接后函数（post-link）:

```js
function addLinkFns(preLinkFn, postLinkFn, attrStart, attrEnd,
  isolateScope, require) {
  if (preLinkFn) {
  //   if (attrStart) {
  //     preLinkFn = groupElementsLinkFnWrapper(preLinkFn, attrStart, attrEnd);
  //   }
  //   preLinkFn.isolateScope = isolateScope;
    preLinkFn.require = require;
  //   preLinkFns.push(preLinkFn);
  }
  if (postLinkFn) {
  //   if (attrStart) {
  //     postLinkFn = groupElementsLinkFnWrapper(postLinkFn, attrStart, attrEnd);
  //   }
  //   postLinkFn.isolateScope = isolateScope;
    postLinkFn.require = require;
    // postLinkFns.push(postLinkFn);
  }
}
```

因为我们现在在节点链接函数调用了这些链接前函数和链接后函数，我们可以检查他们是否有`require`属性。如果有，我们会向链接函数传递第四个参数。这个参数的值将会是一个新函数的返回值，这个新函数称为`getControllers`，下面我们马上就会对这个函数进行定义：

```js
_.forEach(preLinkFns, function(linkFn) {
  linkFn(
  //   linkFn.isolateScope ? isolateScope : scope,
  //   $element,
  //   attrs,
    linkFn.require && getControllers(linkFn.require)
  );
});
// if (childLinkFn) {
//   childLinkFn(scope, linkNode.childNodes);
// }
_.forEachRight(postLinkFns, function(linkFn) {
  linkFn(
    // linkFn.isolateScope ? isolateScope : scope,
    // $element,
    // attrs,
    linkFn.require && getControllers(linkFn.require)
  );
});
```

这个`getControllers`函数需要定义在`applyDirectivesToNode`函数内，这样它才能够访问`controllers`变量，也就是当前节点中存储的（非完全构造）的控制器：

```js
function applyDirectivesToNode(directives, compileNode, attrs) {
  var $compileNode = $(compileNode);
  var terminalPriority = -Number.MAX_VALUE;
  var terminal = false;
  var preLinkFns = [],
    postLinkFns = [],
    controllers = {};
  var newScopeDirective, newIsolateScopeDirective;
  var controllerDirectives;

  function getControllers(require) {

  }
  
  // ...
}
```

这个函数要做的就是找到需要被注入的控制器并返回。如果找不到所需的控制器，它会抛出一个异常：

```js
function getControllers(require) {
  var value;
  if (controllers[require]) {
    value = controllers[require].instance;
  }
  if (!value) {
    throw 'Controller ' + require + ' required by directive, cannot be found!';
  }
  return value;
}
```

注意，在`controllers`变量中保存的都是上一节介绍的“未完全构造”的控制器函数，所以我们需要访问`instance`属性以获取真正的控制器对象，这个对象将会被之后的链接函数接收。到那时才算是被完全构造出来。