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

test/compile_spec.js

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

如果我们从`initializeDirectiveBindings`里的属性 observer 调用`recordChanges`，就能满足这个单元测试了：

```js
case '@':
  attrs.$observe(attrName, function(newAttrValue) {
    var oldValue = destination[scopeName];
    destination[scopeName] = newAttrValue;
    recordChanges(scopeName, destination[scopeName], oldValue);
  });
  if (attrs[attrName]) {
    destination[scopeName] = $interpolate(attrs[attrName])(scope);
  }
  initialChanges[scopeName] =
    new SimpleChange(_UNINITIALIZED_VALUE, destination[scopeName]);
  break;
```

但如果我们在同一个组件上有两个绑定数据，一个是单向绑定数据，另一个是属性绑定，那又会怎么样呢？我们希望在这种情况下，组件的`$onChanges`钩子函数调用时能用一个包含两个绑定数据变化值的对象作为参数：

test/compile_spec.js

```js
it('calls $onChanges once with multiple changes', function() {
  var changesSpy = jasmine.createSpy();
  var attrs;
  var injector = makeInjectorWithComponent('myComponent', {
    bindings: {
      myBinding: '<',
      myAttr: '@'
    },
    controller: function($attrs) {
      this.$onChanges = changesSpy;
      attrs = $attrs;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    $rootScope.aValue = 42;
    var el = $(
      '<my-component my-binding="aValue" my-attr="fourtyTwo"></my-component>'
    );
    $compile(el)($rootScope);
    $rootScope.$apply();
    expect(changesSpy.calls.count()).toBe(1);

    $rootScope.aValue = 43;
    attrs.$set('myAttr', 'fourtyThree');
    $rootScope.$apply();
    expect(changesSpy.calls.count()).toBe(2);
    
    var lastChanges = changesSpy.calls.mostRecent().args[0];
    expect(lastChanges.myBinding.currentValue).toBe(43);
    expect(lastChanges.myBinding.previousValue).toBe(42);
    expect(lastChanges.myAttr.currentValue).toBe('fourtyThree');
    expect(lastChanges.myAttr.previousValue).toBe('fourtyTwo');
  });
});
```

这样是行不通的。原因是我们实际上调用了两次`$onChange`，每次只处理一个变化值。这又是因为我们在`recordChanges`中直接调用`$onChanges`，而没有等到所有变化值都打包好了再调用。我们希望的是，等到本次 digest 循环的所有变化都发生后，再一次性调用`$onChanges`函数。

我们先把`changes`对象从`recordChanges`函数中拿出来（放到`initializeDirectiveBindings`作用域中），这样即使`recordChanges`函数调用之后，`changes`对象还存在。

_src/compile.js_

```js
function initializeDirectiveBindings(scope, attrs, destination, bindings, newScope) {
  // var initialChanges = {};
  var changes;

  function recordChanges(key, currentValue, previousValue) {
    if (destination.$onChanges && currentValue !== previousValue) {
      changes = changes || {};
      // changes[key] = new SimpleChange(previousValue, currentValue);
      // destination.$onChanges(changes);
    }
  }

  // ...

}
```

然后，我们会加入一个新的辅助函数叫`flushOnChanges`，它会把收集到的变化值作为调用`$onChanges`的参数，然后把`changes`对象进行清空。

```js
function flushOnChanges() {
  try {
    destination.$onChanges(changes);
  } finally {
    changes = null;
  }
}
```

我们会让`flushOnChanges`进行定时延迟执行。无论我们何时记录到变化值都会这样做，但如果当前已经在定时执行了，我们就不再重复调用了。我们可以利用一个辅助变量`willflushOnChanges`来进行跟踪。对于具体用什么进行定时任务，我们可以使用`$rootScope.$$postDigest`。这意味着`$onChanges`会在下一个（或当前的）digest 中调用。

_src/compile.js_

```js
function initializeDirectiveBindings(scope, attrs, destination, bindings, newScope) {
  var initialChanges = {};
  var changes;
  var willFlushOnChanges = false;

  function recordChanges(key, currentValue, previousValue) {
    if (destination.$onChanges && currentValue !== previousValue) {
      changes = changes || {};
      changes[key] = new SimpleChange(previousValue, currentValue);
      if (!willFlushOnChanges) {
        $rootScope.$$postDigest(flushOnChanges);
        willFlushOnChanges = true;
      }
    }
  }

  function flushOnChanges() {
    try {
      destination.$onChanges(changes);
    } finally {
      changes = null;
    }
  }

  // ...

}
```

我们需要在`flushOnChanges`函数中把`willFlushChanges`进行重置，这样我们之后才能再次设置定时任务。

```js
function flushOnChanges() {
  try {
    destination.$onChanges(changes);
  } finally {
    changes = null;
    willFlushOnChanges = false;
  } 
}
```

这样就能让目前的测试集正常工作了，但还是有一些问题。首先，如果我们在控制器的`$onChanges`方法中有代码会修改绑定了的属性，这些改变并没有被接收到。我们可以看到在这个单元测试中，组件模板并没有包含我们希望有的内容：

_test/compile_spec.js_

```js
it('runs $onChanges in a digest', function() {
  var changesSpy = jasmine.createSpy();
  var injector = makeInjectorWithComponent('myComponent', {
    bindings: {
      myBinding: '<'
    },
    controller: function() {
      this.$onChanges = function() {
        this.innerValue = 'myBinding is ' + this.myBinding;
      };
    },
    template: '{{ $ctrl.innerValue }}'
  });
  injector.invoke(function($compile, $rootScope) {
    $rootScope.aValue = 42;
    var el = $('<my-component my-binding="aValue"></my-component>');
    $compile(el)($rootScope);
    $rootScope.$apply();
    
    $rootScope.aValue = 43;
    $rootScope.$apply();

    expect(el.text()).toEqual('myBinding is 43');
  });
});
```

出现这种情况是由于我们使用了`$$postDigest`，而`$$postDigest`实际上是在 digest 以外执行的。我们在这里产生的变化不会马上生效直到下一个 digest 发生，但我们不知道究竟是在什么时候。作为应用开发者，我们希望所有的控制器代码，包括`$onChanges`，会在一个 digest 中全部执行，这样我们就不用再调用一次`$scope.$apply`。所以，我们要做的是在调用`$onChanges`时启动另一个 digest。

_src/compile.js_

```js
function flushOnChanges() {
  try {
    $rootScope.$apply(function() {
      destination.$onChanges(changes);
    });
  } finally {
    changes = null;
    willFlushOnChanges = false;
  }
}
```

另一个问题发生在一次 digest 中同一个绑定数据被修改多次的情况。变化值确实能被记录，但我们就无法跟踪到 digest 之前发生的旧值是什么了，因为每次值的改变都会覆写上一个值。

_test/compile_spec.js_

```js
it('keeps first value as previous for $onChanges when multiple changes', function() {
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
    
    $rootScope.aValue = 43;
    $rootScope.$watch('aValue', function() {
      if ($rootScope.aValue !== 44) {
        $rootScope.aValue = 44;
      }
    });
    $rootScope.$apply();
    expect(changesSpy.calls.count()).toBe(2);

    var lastChanges = changesSpy.calls.mostRecent().args[0];
    expect(lastChanges.myBinding.currentValue).toBe(44);
    expect(lastChanges.myBinding.previousValue).toBe(42);
  });
});
```

在`recordChanges`函数中，我们需要检查是否已经有对当前 key 的 change object。如果有，我们就把它的旧值抓取下来作为我们新的改变记录，并丢弃中间的变化值。在`$onChanges`中，无论在 digest 中绑定数据改变了多少次，我们将看到第一个旧值和最后一个新值。

_src/compile.js_

```js
function recordChanges(key, currentValue, previousValue) {
  // if (destination.$onChanges && currentValue !== previousValue) {
  //   changes = changes || {};
    if (changes[key]) {
      previousValue = changes[key].previousValue;
    }
  //   changes[key] = new SimpleChange(previousValue, currentValue);
  //   if (!willFlushOnChanges) {
  //     $rootScope.$$postDigest(flushOnChanges);
  //     willFlushOnChanges = true;
  //   }
  // }
}
```

下一个要解决的问题，可能也是最严重的一个问题，因为它会影响到性能。留意每当我们 flush 改变的队列时，我们调用`$scope.$apply()`的方式。这在每个跟踪`$onChanges`中的变化的组件中都调用了一次。想一想，如果一个应用在视图上有 50 个这样的组件，这种情况在真实的应用发生也是挺正常。那么，在这个应用中，我们就需要在一次 digest 中额外启动多达 50 个 digest。还有，按照我们之前学到的，每个`$apply`将会把应用中所有的变化侦测代码都运行一遍。这就是现在我们面临的严重的性能问题。

我们想要实现的效果是，无论有多少个组件存在，我们也只会启动一个 digest。我们可以在同一个 digest 中同时为所有组件调用`$onChanges`。

_test/compile_spec.js_

```js
it('runs $onChanges for all components in the same digest', function() {
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.component('first', {
      bindings: { myBinding: '<' },
      controller: function() {
        this.$onChanges = function() {};
      }
    });
    $compileProvider.component('second', {
      bindings: { myBinding: '<' },
      controller: function() {
        this.$onChanges = function() {};
      }
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var watchSpy = jasmine.createSpy();
    $rootScope.$watch(watchSpy);

    $rootScope.aValue = 42;
    var el = $('<div>' +
      '<first my-binding="aValue"></first>' +
      '<second my-binding="aValue"></second>' +
      '</div>');
    $compile(el)($rootScope);
    $rootScope.$apply();
    // Dirty watches always cause a second digest, hence 2
    expect(watchSpy.calls.count()).toBe(2);
    
    $rootScope.aValue = 43;
    $rootScope.$apply();
    // Two more because of dirty watches,
    // plus *one* more for onchanges
    expect(watchSpy.calls.count()).toBe(5);
  });
});
```

我们下一步要做的是把一些用于对变化进行检测的代码拿到外面，这样我们就可以一次性把所有变化都清理掉。但首先，让我们先将`flushOnChanges`中对`$onChanges`的调用放到一个独立的函数中，这个函数叫`triggerOnChanges`。

_src/compile.js_

```js
function triggerOnChanges() {
  try {
    destination.$onChanges(changes);
  } finally {
    changes = null;
  }
}

function flushOnChanges() {
  $rootScope.$apply(function() {
    triggerOnChanges();
    willFlushOnChanges = false;
  });
}
```

然后，我们要将`willFlushOnChanges`布尔值标识放到一个叫`onChangesQueue`的数组中。我们主要是想把多个`$onChanges`触发器放到一个调用队列里面去。每当我们记录了一个变化，如果这个数组形式的队列还没被初始化，我们会把它初始化为一个空数组。同时，我们会执行一个定时任务，约定在之后某个时间统一对变化进行处理。

```js
function initializeDirectiveBindings(scope, attrs, destination, bindings, newScope) {
  var initialChanges = {};
  var changes;
  var onChangesQueue;

  function recordChanges(key, currentValue, previousValue) {
    if (destination.$onChanges && currentValue !== previousValue) {
      if (!onChangesQueue) {
        onChangesQueue = [];
        $rootScope.$$postDigest(flushOnChanges);
      }
      changes = changes || {};
      if (changes[key]) {
        previousValue = changes[key].previousValue;
      }
      changes[key] = new SimpleChange(previousValue, currentValue);
    }
  }

  // ...

}
```

然后我们会逐个将变化的触发函数加入到队列中：

```js
function recordChanges(key, currentValue, previousValue) {
  if (destination.$onChanges && currentValue !== previousValue) {
    // if (!onChangesQueue) {
    //   onChangesQueue = [];
    //   $rootScope.$$postDigest(flushOnChanges);
    // }
    if (!changes) {
      changes = {};
      onChangesQueue.push(triggerOnChanges);
    }
    // if (changes[key]) {
    //   previousValue = changes[key].previousValue;
    // }
    // changes[key] = new SimpleChange(previousValue, currentValue);
  }
}
```

也就是说，最后`onChangesQueue`会变成一个待调用函数的集合。这也是我们在`flushOnChanges`要做的事情：

```js
function flushOnChanges() {
  $rootScope.$apply(function() {
    _.forEach(onChangesQueue, function(onChangesHook) {
      onChangesHook();
    });
    onChangesQueue = null;
  }); 
}
```

要做的最后一件事是把`onChangeQueue`变量和`flushOnChanges`函数放到`initializeDirectiveBindings`函数之外，就放在`$CompileProvider`的`$get`方法里，这样它们会被整个应用分享：

```js
this.$get = ['$injector', '$parse', '$controller', '$rootScope', '$http',
  '$interpolate',
  function($injector, $parse, $controller, $rootScope, $http, $interpolate) {
    
    var onChangesQueue;

    function flushOnChanges() {
      $rootScope.$apply(function() {
        _.forEach(onChangesQueue, function(onChangesHook) {
          onChangesHook();
        });
        onChangesQueue = null;
      });
    }

    // ...
  
  }
];
```

而`changes`对象和`triggerOnChanges`会待在它们原来的地方，因为这两个变量特定于每一个组件的。

因此，目前将会发生的情况是应用中的任何一个组件在何时何处需要调用`$onChanges`，程序会初始化`onChangesQueue`队列，并将组件自己的`triggerOnChanges`函数放到里面去。然后，所有其他同样要调用`$onChanges`的组件会把各自的`triggerOnChanges`放到这个队列中。当 digest 结束后，当要处理队列是，就会把所有触发器会进行一个批量执行。

但还有一件事情。如果我们有一个`$onChanges`方法会引起其他组件的绑定数据发生改变该怎么办呢？单个组件这么做是没有问题，但如果我们要把两个组件放到一起，而且两个组件形成了相互改变关联属性的情况，就会产生循环问题。像下面的这个测试用例会产生一个无限循环。

_test/compile_spec.js_

```js
it('has a TTL for $onChanges', function() {
  var injector = createInjector(['ng', function($compileProvider) {
    $compileProvider.component('myComponent', {
      bindings: {
        input: '<',
        increment: '='
      },
      controller: function() {
        this.$onChanges = function() {
          if (this.increment) {
            this.increment = this.increment + 1;
          }
        };
      }
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var watchSpy = jasmine.createSpy();
    $rootScope.$watch(watchSpy);

    var el = $('<div>' +
      '<my-component input="valueOne" increment="valueTwo"></my-component>' +
      '<my-component input="valueTwo" increment="valueOne"></my-component>' +
      '</div>');
    $compile(el)($rootScope);
    $rootScope.$apply();
    $rootScope.valueOne = 42;
    $rootScope.valueTwo = 42;
    $rootScope.$apply();
    expect($rootScope.valueOne).toBe(51);
    expect($rootScope.valueTwo).toBe(51);
  });
});
```