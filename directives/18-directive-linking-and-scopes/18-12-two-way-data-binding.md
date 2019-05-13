### 双向数据绑定（Two-Way Data Binding）

双向数据绑定跟单向数据绑定很类似。之后我们会介绍两者之间几个重要的差异点，但我们先把双向数据绑定中跟单向数据绑定相同的特性先实现出来。

对于双向数据绑定，最简单的配置方式是在作用域定义对象中用`'='`符号来进行指定：

_test/compile_spec.js_

```js
it('allows binding two-way expression to isolate scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        anAttr: '='
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive an-attr="42"></div>');
    $compile(el)($rootScope);

    expect(givenScope.anAttr).toBe(42);
  });
})
```

我们也可以对双向绑定的属性取一个别名：

```js
it('allows aliasing two-way expression attribute on isolate scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '=theAttr'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive the-attr="42"></div>');
    $compile(el)($rootScope);
    
    expect(givenScope.myAttr).toBe(42);
  });
});
```

像单向数据绑定一样，双向数据绑定的属性也是会被 watch 的：

```js
it('watches two-way expressions', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '='
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="parentAttr + 1"></div>');
    $compile(el)($rootScope);
    
    $rootScope.parentAttr = 41;
    $rootScope.$digest();
    expect(givenScope.myAttr).toBe(42);
  });
});
```

属性值也可以是可选的，我们可以使用`=?`的语法：

```js
it('does not watch optional missing two-way expressions', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '=?'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);
    expect($rootScope.$$watchers.length).toBe(0);
  });
});
```

上面这几个测试用例，我们只需要在之前使用的属性绑定正则表达式中加入对`=`符号的支持即可。

_src/compile.js_

```js
function parseIsolateBindings(scope) {
  var bindings = {};
  _.forEach(scope, function(defnition, scopeName) {
    var match = defnition.match(/\s*([@<=])(\??)\s*(\w*)\s*/);
    // ...
```

然后，我们把之前实现单向数据绑定的代码复制到双向数据绑定的处理逻辑中：

```js
_.forEach(
  // newIsolateScopeDirective.$$isolateBindings,
  // function(defnition, scopeName) {
  //   var attrName = defnition.attrName;
    var parentGet, unwatch;
    // switch (defnition.mode) {
    //   case '@':
    //     attrs.$observe(attrName, function(newAttrValue) {
    //       isolateScope[scopeName] = newAttrValue;
    //     });
    //     if (attrs[attrName]) {
    //       isolateScope[scopeName] = attrs[attrName];
    //     }
    //     break;
    //   case '<':
    //     if (defnition.optional && !attrs[attrName]) {
    //       break;
    //     }
        parentGet = $parse(attrs[attrName]);
        // isolateScope[scopeName] = parentGet(scope);
        unwatch = scope.$watch(parentGet, function(newValue) {
        //   isolateScope[scopeName] = newValue;
        // });
        // isolateScope.$on('$destroy', unwatch);
        // break;
      case '=':
        if (defnition.optional && !attrs[attrName]) {
          break;
        }
        parentGet = $parse(attrs[attrName]);
        isolateScope[scopeName] = parentGet(scope);
        unwatch = scope.$watch(parentGet, function(newValue) {
          isolateScope[scopeName] = newValue;
        });
        isolateScope.$on('$destroy', unwatch);
        break;
    // }
  });
```

但如果这就是双向数据绑定的全部内容，那双向数据绑定也没有什么新意，是吧？现在，我们开始介绍双向数据绑定的“双向”特性：当我们在独立作用域上声明一个双向数据绑定属性，它也会对父作用域的对应绑定产生影响。这是我们之前并未接触过的。下面就是一个例子：

_test/compile_spec.js_

```js
it('allows assigning to two-way scope expressions', function() {
  var isolateScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '='
      },
      link: function(scope) {
        isolateScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="parentAttr"></div>');
    $compile(el)($rootScope);
    
    isolateScope.myAttr = 42;
    $rootScope.$digest();
    expect($rootScope.parentAttr).toBe(42);
  });
});
```

在这个测试中我们会把一个独立作用域上的属性`myAttr`和父作用域上的`parentAttr`进行绑定。我们对子作用域的属性赋值并运行一个 digest 循环，测试对应在父作用域上的属性是否也会同步更新。

一旦数据能够实现双向联系，优先级就会成为一个问题：如果父作用域和子作用域属性都在同一次 digest 循环中都改变了该怎么办？哪个属性的值变更会成为两者最后的计算结果？在 Angular，父元素的属性值变更会被优先使用：

_test/compile_spec.js_

```js
it('gives parent change precedence when both parent and child change', function() {
  var isolateScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '='
      },
      link: function(scope) {
        isolateScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive my-attr="parentAttr"></div>');
    $compile(el)($rootScope);

    $rootScope.parentAttr = 42;
    isolateScope.myAttr = 43;
    $rootScope.$digest();
    expect($rootScope.parentAttr).toBe(42);
    expect(isolateScope.myAttr).toBe(42);
  });
});
```

这就是双向数据绑定运行的基本原则。下面我们就来实现它，它并不复杂，但有几个小细节我们需要先解决。

首先在值变更时，我们需要在单纯的观察者机制的基础上加入更多控制。为了让事情变得更简单清晰，我们只注册一个 watch 函数而忽略对应的监听处理函数。我们仅依赖 watch 函数在每次 digest 函数中会被调用的特性，并在 watch 函数中加入自己的处理逻辑：

_src/compile.js_

```js
case '=':
  // if (defnition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  // isolateScope[scopeName] = parentGet(scope);
  var parentValueWatch = function() {
    var parentValue = parentGet(scope);
    if (isolateScope[scopeName] !== parentValue) {
      isolateScope[scopeName] = parentValue;
    }
    return parentValue;
  };
  unwatch = scope.$watch(parentValueWatch);
  // isolateScope.$on(‘$destroy’, unwatch);
  // break;
```

当前实现也只是让旧的单元测试通过而已，而最新的单元测试仍没通过，但是现在的代码更适合我们实现双向数据绑定的“双向”特性。

现在我们只检查了当前的属性值是否不同于独立作用域的属性值，但没有检查到这个变化实际上是发生自哪里的。为了能检查获取这个信息，我们需要引进一个新的变量`lastValue`，它会一直保存最近一次 digest 循环后在父作用域上的属性值：

_src/compile.js_

```js
case '=':
  // if (defnition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  var lastValue = isolateScope[scopeName] = parentGet(scope);
  // var parentValueWatch = function() {
  //   var parentValue = parentGet(scope);
  //   if (isolateScope[scopeName] !== parentValue) {
      if (parentValue !== lastValue) {
        // isolateScope[scopeName] = parentValue;
      }
    // }
    lastValue = parentValue;
    return lastValue;
  // };
  // unwatch = scope.$watch(parentValueWatch);
  // isolateScope.$on('$destroy', unwatch);
  // break;
```

这个变量的作用就是让我们能分辨以下情况：独立作用域属性的当前值不同于父作用域上的属性值，但父作用域属性值等于`lastValue`，这就说明属性值是在独立作用域上更改的，我们需要将变更同步给父作用域属性：

_src/compile.js_

```js
case '=':
  // if (defnition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  // var lastValue = isolateScope[scopeName] = parentGet(scope);
  // var parentValueWatch = function() {
  //   var parentValue = parentGet(scope);
  //   if (isolateScope[scopeName] !== parentValue) {
  //     if (parentValue !== lastValue) {
  //       isolateScope[scopeName] = parentValue;
      // } 
      else {

      }
  //   }
  //   lastValue = parentValue;
  //   return lastValue;
  // };
  // unwatch = scope.$watch(parentValueWatch);
  // break;
```

我们该怎么更新父作用域上的属性呢？我们之前开发表达式的时候，曾经讲到有一些表达式是可以从外部进行赋值的，这意味着它们不仅可以通过计算获取结果值，还可以利用表达式的`assign`函数把一个新值赋值给表达式。这就是我们用新值来更新父作用域属性要用到的工具：

_src/compile.js_

```js
case '=':
  // if (defnition.optional && !attrs[attrName]) {
  //   break;
  // }
  // parentGet = $parse(attrs[attrName]);
  // var lastValue = isolateScope[scopeName] = parentGet(scope);
  // var parentValueWatch = function() {
  //   var parentValue = parentGet(scope);
  //   if (isolateScope[scopeName] !== parentValue) {
  //     if (parentValue !== lastValue) {
  //       isolateScope[scopeName] = parentValue;
  //     } else {
        parentValue = isolateScope[scopeName];
        parentGet.assign(scope, parentValue);
  //     }
  //   }
  //   lastValue = parentValue;
  //   return lastValue;
  // };
  // unwatch = scope.$watch(parentValueWatch);
  // isolateScope.$on(‘$destroy’, unwatch);
  // break;
```