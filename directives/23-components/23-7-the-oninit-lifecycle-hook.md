### 生命周期钩子函数 $onInit（The $onInit Lifecycle Hook）

本章的第二部分都是跟定义控制器生命周期函数有关的。这些方法都是控制器可能会用到的，它们会在组件的生命周期中某个特定时间点被调用，让应用开发者有机会在这些特定时间点加入一些自定义的逻辑。

在 Angular 1.5 之前的版本都还没有这些生命周期钩子。之前只有指令的 prelink、 postlink 函数和控制器构造函数能起到类似的作用。但事实证明，这些函数的作用有限，更不要说搞清楚它们之间调用的顺序。

React 和 Angular 2 已经有一个定义明确的组件生命周期，Angular 1.5 的组件生命周期在很大程度上是从这两个框架借鉴过来的。组件生命周期也确实让控制器的代码更易理解和管理，现在我们就一起来看看它是怎么实现的。

第一个要说的生命周期钩子是`$onInit`，它在组件元素完全初始化后调用。它至少有两个用处：

- 它被调用的时候，元素上的所有指令都已经初始化好了，这意味着我们`required`的所有东西都处于可用状态了。如果我们需要用到引入的控制器，那在这个钩子函数中进行初始化就是安全的。
- 我们可以将控制器的初始化逻辑放在这里，而不用放到控制器构造函数中，这样对控制器进行单元测试时就更简单。我们不再需要使用“非完全构造的控制器”的 hack 来定义需要在初始化阶段进行定义的控制器绑定数据。

我们来再理一下当元素上同时拥有一个组件和一个指令时的事件发生顺序。首先，我们希望这两者的控制器构造函数都已经调用完成了，然后才是调用`$onInit`方法。这两个过程都结束后才会调用指令的链接函数：

_test/compile_spec.js_

```js
describe('lifecycle', function() {
  
  it('calls $onInit after all ctrls created before linking', function() {
    var invocations = [];
    var injector = createInjector(['ng', function($compileProvider) {
      $compileProvider.component('first', {
        controller: function() {
          invocations.push('first controller created');
          this.$onInit = function() {
            invocations.push('first controller $onInit');
          };
        }
      });
      $compileProvider.directive('second', function() {
        return {
          controller: function() {
            invocations.push('second controller created');
            this.$onInit = function() {
              invocations.push('second controller $onInit');
            };
          },
          link: {
            pre: function() { invocations.push('second prelink'); },
            post: function() { invocations.push('second postlink'); }
          }
        };
      });
    }]);
    injector.invoke(function($compile, $rootScope) {
      var el = $('<first second></first>');
      $compile(el)($rootScope);
      expect(invocations).toEqual([
        'first controller created',
        'second controller created',
        'first controller $onInit',
        'second controller $onInit',
        'second prelink',
        'second postlink'
      ]);
    });
  });

});
```