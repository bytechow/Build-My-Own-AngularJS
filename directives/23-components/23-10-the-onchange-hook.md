### 生命周期钩子函数——$onChanges（The $onChanges Hook）

最后要讲的生命周期钩子函数是`$onChanges`。这个钩子函数也许是所有钩子函数中最有用的，但也是目前为止开发起来最为复杂的一个钩子函数。

它的主要思路是在组件绑定数据发生改变时发出通知。也就是说，当组件中的一个或多个输入项发生改变时，应用开发者可以加入一些处理逻辑，如完成派生值的计算或其他需要在变化时处理的事情。

这个功能点也是之前并没有给出比较理想的解决方案的。在 Angular 1.5 版本出来之前，我们可以在指令控制器中添加`$watch`，然后在 watch 的 listener 函数中实现这一功能：

```js
$scope.$watch('ctrl.someInput', function(newValue) {
  ctrl.derivedValue = deriveValue(newValue);
});
```

这种 API 不仅不方便，而且会带来性能的损耗。我们对已经被指令编译器 watch 的东西再加一个 watcher，这真是没什么必要。对于`$onChanges`，它实际上是不必要的，因为现在框架已经可以告诉我们发生了变化。

`$onChanges`第一次调用是发生在组件被初始化后。它会提供组件所有属性和单向绑定数据的初始值。

_test/compile_spec.js_

```js
it('calls $onChanges with all bindings during init', function() {
  var changesSpy = jasmine.createSpy();
  var injector = makeInjectorWithComponent('myComponent', {
    bindings: {
      myBinding: '<',
      myAttr: '@'
    },
    controller: function() {
      this.$onChanges = changesSpy;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component my-binding="42" my-attr="43"></my-component>');
    $compile(el)($rootScope);
    expect(changesSpy).toHaveBeenCalled();
     var changes = changesSpy.calls.mostRecent().args[0];
    expect(changes.myBinding.currentValue).toBe(42);
    expect(changes.myBinding.isFirstChange()).toBe(true);
    expect(changes.myAttr.currentValue).toBe('43');
    expect(changes.myAttr.isFirstChange()).toBe(true);
  }); 
});
```

注意，传递给`$onChange`的变化值参数的结构是这样的：它会包含与绑定数据相匹配的 key。对每一个绑定数据来说，它都包含一个`currentValue`的 key，还有一个会返回`true`结果值的`isFirstValue()`函数。

关于`$onChanges`，还有一样东西需要注意：双向绑定数据的变化是无法捕捉到的。要使用`$onChanges`，应用开发者就需要习惯使用单项数据绑定：

_test/compile_spec.js_

```js
it('does not call $onChanges for two-way bindings', function() {
  var changesSpy = jasmine.createSpy();
  var injector = makeInjectorWithComponent('myComponent', {
  bindings: {
      myBinding: '=',
    },
    controller: function() {
      this.$onChanges = changesSpy;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component my-binding="42"></my-component>');
    $compile(el)($rootScope);
    expect(changesSpy).toHaveBeenCalled();
    expect(changesSpy.calls.mostRecent().args[0].myBinding).toBeUndefined();
  }); 
});
```

我们会将在`initializeDirectiveBindings`函数中的初始变化值进行收集，最后返回它们：

_src/compile.js_

```js
function initializeDirectiveBindings(scope, attrs, destination, bindings, newScope)
{
  var initialChanges = {};
  _.forEach(bindings, function(definition, scopeName) {
    // ...
  });
  return initialChanges;
}
```

但我们需要收集哪些值呢？其实，我们已经能在测试用例中看到 change 对象的大致轮廓：里面会有当前值、变化前的值和一个`isFirstChange()`方法。它们实际上是在`compile.js`中的一个小小的构造函数中构造出来的，这个构造函数叫做`SimpleChange`。我们会在这个文件的顶层作用域定义这个构造函数：

_src/compile.js_

```js
function SimpleChange(previous, current) {
  this.previousValue = previous;
  this.currentValue = current;
}
```

这个构造函数会对`isFirstChange()`方法进行定义，这个方法会将变化前的值与一个叫`_UNINITIANLIZED_VALUE`的常量进行对比。

```js
// function SimpleChange(previous, current) {
//   this.previousValue = previous;
//   this.currentValue = current;
// }
SimpleChange.prototype.isFirstChange = function() {
  return this.previousValue === _UNINITIALIZED_VALUE;
};
```

这个常量也会在顶层作用域中进行定义。它具体的值倒不是非常重要，因为这个值只是用于检查这次改变是不是首次改变。我们要注意的是，这个值只能与自身相等。

```js
function UNINITIALIZED_VALUE() { }
var _UNINITIALIZED_VALUE = new UNINITIALIZED_VALUE();
```

现在，我们可以在`initializeDirectiveBindings`函数中生成初始的变化值。注意，我们只会对属性绑定和单向数据绑定进行初始化，不包括双向数据绑定：

```js
case '@':
  // attrs.$observe(attrName, function(newAttrValue) {
  //   destination[scopeName] = newAttrValue;
  // });
  // if (attrs[attrName]) {
  //   destination[scopeName] = $interpolate(attrs[attrName])(scope);
  // }
  initialChanges[scopeName] =
    new SimpleChange(_UNINITIALIZED_VALUE, destination[scopeName]);
  break;
case '<':
  // if (definition.optional && !attrs[attrName]) {
  //   break; 
  // }
  // parentGet = $parse(attrs[attrName]);
  // destination[scopeName] = parentGet(scope);
  // unwatch = scope.$watch(parentGet, function(newValue) {
  //   destination[scopeName] = newValue;
  // });
  // newScope.$on('$destroy', unwatch);
  initialChanges[scopeName] =
    new SimpleChange(_UNINITIALIZED_VALUE, destination[scopeName]);
  break;
```

由于我们对每一个控制器都调用了`initializeDirectiveBindings`，我们需要把抓取到的返回值存放在`initialChanges`中，这个`initialChanges`将会被添加为内部的控制器对象的属性，以便稍后使用。

_src/compile.js_

```js
// var scopeDirective = newIsolateScopeDirective || newScopeDirective;
if (scopeDirective && controllers[scopeDirective.name]) {
  controllers[scopeDirective.name].initialChanges = initializeDirectiveBindings(
      scope,
      attrs,
      controllers[scopeDirective.name].instance,
      scopeDirective.$$bindings.bindToController,
      isolateScope
  ); 
}
```

我们会在调用`$onInit`之后使用这些初始变化值。如果在这个时间点控制器中注册了`$onChanges`钩子，我们就会利用`initialChanges`对这个钩子函数进行第一次调用：

_src/compile.js_

```js
_.forEach(controllers, function(controller) {
  // var controllerInstance = controller.instance;
  // if (controllerInstance.$onInit) {
  //   controllerInstance.$onInit();
  // }
  if (controllerInstance.$onChanges) {
    controllerInstance.$onChanges(controller.initialChanges);
  }
  // if (controllerInstance.$onDestroy) {
  //   (newIsolateScopeDirective ? isolateScope : scope).$on('$destroy', function() {
  //     controllerInstance.$onDestroy();
  //   });
  // }
});
```

接下来，我们就要实现`$onChanges`的实际用途——在发生变化时进行调用，而不仅仅是在组件初始化时调用。假设有一个组件包含了一个单向数据绑定的输入框，我们对输入框的值进行改变的时候，我们会希望这个组件的`$onChanges()`会在下一个 digest 中调用。

_test/compile_spec.js_

```js
it('calls $onChanges when binding changes', function() {
  var changesSpy = jasmine.createSpy();
  var injector = makeInjectorWithComponent('myComponent', {
    bindings: {
      myBinding: '<'
    },
    controller: function() {
      this.$onChanges = changesSpy;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    $rootScope.aValue = 42;
    var el = $('<my-component my-binding="aValue"></my-component>');
    $compile(el)($rootScope);
    $rootScope.$apply();

    expect(changesSpy.calls.count()).toBe(1);
    
    $rootScope.aValue = 43;
    $rootScope.$apply();
    expect(changesSpy.calls.count()).toBe(2);
    
    var lastChanges = changesSpy.calls.mostRecent().args[0];
    expect(lastChanges.myBinding.currentValue).toBe(43);
    expect(lastChanges.myBinding.previousValue).toBe(42);
    expect(lastChanges.myBinding.isFirstChange()).toBe(false);
  });
});
```

注意这次变化记录中我们也已经有一个旧值和当前值，而且这次的变化我们就不能认为是第一次变化了，也就是说调用`isFirstChange()`函数的返回值会是`false`。

在`initalizeDirectiveBindings`中，我们会对单向数据绑定设置一个 watcher。这就是我们能够获知到变化发生的关键。这里我们会调用一个叫`recordChanges`的帮助函数。我们会向这个函数传入绑定数据的名称，还有绑定数据的旧值和新值。有了这些信息，我们就能构建出`$onChanges`所需要的东西了。

_src/compile.js_

```js
case '<':
  // if (definition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  // destination[scopeName] = parentGet(scope);
  unwatch = scope.$watch(parentGet, function(newValue) {
    var oldValue = destination[scopeName];
    // destination[scopeName] = newValue;
    recordChanges(scopeName, destination[scopeName], oldValue);
  });
  // newScope.$on('$destroy', unwatch);
  // initialChanges[scopeName] =
  //   new SimpleChange(_UNINITIALIZED_VALUE, destination[scopeName]);
  // break;
```

我们会在`initializeDirectiveBindings`的内部中定义`recordChanges`函数，形成闭包：

```js
function initializeDirectiveBindings(scope, attrs, destination, bindings, newScope) {
  // var initialChanges = {};

  function recordChanges(key, currentValue, previousValue) {

  }
  
  // _.forEach(bindings, function(definition, scopeName) {
  //   var attrName = definition.attrName;
  //   var parentGet, unwatch;
  //   switch (definition.mode) {
  //     // ...
  //   }
  // });
  // return initialChanges;
}
```

这个函数中我们可以做的最简单、合理的一件事就是检查控制器（通过`destination`变量能访问到）是否有`$onChanges`方法，并且这个值是否真的改变了。如果这两个条件都成立，我们就会创建一个变化对象，并以它作为参数直接调用`$onChanges`。这就满足我们单元测试的需要了：

```js
function recordChanges(key, currentValue, previousValue) {
  if (destination.$onChanges && currentValue !== previousValue) {
     var changes = {};
     changes[key] = new SimpleChange(previousValue, currentValue);
     destination.$onChanges(changes);
  }
}
```

我们来看看这种方式怎么对属性绑定也生效。当我们对一个已经进行了绑定的属性进行`$set`时，我们希望`$onChanges`会在下一次 digest 中调用。

```js
it('calls $onChanges when attribute changes', function() {
  var changesSpy = jasmine.createSpy();
  var attrs;
  var injector = makeInjectorWithComponent('myComponent', {
    bindings: {
      myAttr: '@'
    },
    controller: function($attrs) {
      this.$onChanges = changesSpy;
      attrs = $attrs;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component my-attr="42"></my-component>');
    $compile(el)($rootScope);
    $rootScope.$apply();
    
    expect(changesSpy.calls.count()).toBe(1);
    
    attrs.$set('myAttr', '43');
    $rootScope.$apply();
    expect(changesSpy.calls.count()).toBe(2);
    
    var lastChanges = changesSpy.calls.mostRecent().args[0];
    expect(lastChanges.myAttr.currentValue).toBe('43');
    expect(lastChanges.myAttr.previousValue).toBe('42');
    expect(lastChanges.myAttr.isFirstChange()).toBe(false);
  });
});
```