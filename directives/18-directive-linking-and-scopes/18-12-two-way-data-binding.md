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

但如果这就是双向数据绑定的全部内容，那双向数据绑定也没有什么新意，是吧？现在，我们开始介绍双向数据绑定的“双向”特性：当我们在独立作用域上声明一个双向数据绑定属性，它也会对父作用域的对应绑定产生影响。这是我们之前并未解除过的。下面就是一个例子：

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