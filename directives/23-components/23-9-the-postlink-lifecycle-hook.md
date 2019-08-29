### 生命周期钩子函数——$postLink（The $postLink Lifecycle Hook）

有时你希望在所有子元素都初始化并进行链接后的时间点执行一些逻辑。举个例子，你可以会在这个时间点访问子元素的 DOM。

对于指令来说，我们可以利用 postlink 函数完成这个需求，这是最简单直接的。但对于无法定义链接函数的组件来说，这种方式就行不通了。代替链接函数在组件中实现这一功能的是`$postLink`钩子函数，这也是唯一可以接触到组件的这个生命周期的方法。

_test/compile_spec.js_

```js
it('calls $postLink after all linking is done', function() {
  var invocations = [];
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.component('first', {
      controller: function() {
        this.$postLink = function() {
          invocations.push('first controller $postLink');
        };
      }
    });
    $compileProvider.directive('second', function() {
      return {
        controller: function() {
          this.$postLink = function() {
            invocations.push('second controller $postLink');
          };
        },
        link: function() { invocations.push('second postlink'); }
      };
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<first><second></second></first>');
    $compile(el)($rootScope);
    expect(invocations).toEqual([
      'second postlink',
      'second controller $postLink',
      'first controller $postLink'
    ]);
  });
});
```

这里我们可以发现`$postLink`钩子函数会在 postlink 函数之后调用，同时这些钩子函数也是从 DOM 树的下层元素逐级往上冒泡的。这就是我们在链接后希望发生的事情。

我们需要在另一个对控制器的遍历中进行这个钩子函数的调用。这个遍历过程会放在节点链接函数的最后，在对 postlink 函数的反向遍历之后。 

_src/compile.js_

```js
// _.forEachRight(postLinkFns, function(linkFn) {
//   linkFn(
//     linkFn.isolateScope ? isolateScope : scope,
//     $element,
//     attrs,
//     linkFn.require && getControllers(linkFn.require, $element),
//     scopeBoundTranscludeFn
//   );
// });

_.forEach(controllers, function(controller) {
  var controllerInstance = controller.instance;
  if (controllerInstance.$postLink) {
    controllerInstance.$postLink();
  }
});
```