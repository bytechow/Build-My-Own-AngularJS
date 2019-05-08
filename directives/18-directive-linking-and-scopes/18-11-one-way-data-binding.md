### 单向数据绑定（One-Way Data Binding）

属性绑定常常能起到不小用处，而独立作用域绑定最常用于数据绑定：让独立作用域上的作用域属性和其父作用域上解析的表达式进行关联。

数据绑定也允许父子作用域之间进行像普通继承作用域一样的数据共享，但其中有几种重要的区别：

|继承作用域|独立作用域|
|--|--|
|父作用域的所有属性都会传递给子作用域|只有在表达式中提及到的属性才会被共享|
|父子属性是一一对应的关系|子作用域属性在父作用域上可能没有对应属性，但可以使用任何表达式|

Angular 有两种数据绑定的类型：单向数据绑定，可以允许我们往独立作用域传递数值。双向绑定，除了能往独立作用域传递属性值，还可以把属性值的变化反馈给父作用域。

在这两个类型中，应用开发者用得最多的是单向绑定。往下传递数据给指令或组件就是一个典型案例。要用到双向绑定的情况比较少，但我们还是可以在有需要时用到它。

> 现实中大多数应用程序中，双向数据绑定还是会比单向绑定更常见。这是有一个历史原因：单向数据绑定直到 Angular 1.5 版本才被加入，而这时双向绑定已经被广泛应用了。因此，大多数情况下我们很可能会会遇到一个向下的双向数据绑定，而我们实际上可以直接用单向数据绑定代替。

让我们开始探索单向绑定是如何实现的。最简单的单向绑定方式是在作用域定义对象中对属性使用'`<`'符号。这就相当于说：“在父作用域上把这个属性值当作一个 Angular 表达式进行解析”。这个表达式本身不会在作用域定义对象进行定义，但会在指令所对应的 DOM 属性上进行定义。下面就是一个例子，`anAttr`就是通过这种方式进行绑定的，属性值表达式`'42'`，将会被解析为数字`42`：

_test/compile_spec.js_

```js
it('allows binding expression to isolate scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        anAttr: '<'
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
});
```

我们希望对应的作用域属性会在我们链接了指令后确实是存在的。

作为一种属性的绑定，你可以对要绑定的属性起一个别名，这样作用域属性不用一定要与 DOM 属性名称一致。绑定方式跟之前的属性绑定一致：

_test/compile_spec.js_

```js
it('allows aliasing expression attribute on isolate scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '<theAttr'
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

数据绑定中的表达式当然不仅仅只支持像`42`这种常量字面量。当我们需要引用到父作用域上的属性时这个绑定的机制才能展现出真正的价值。这也是我们可以将数据从父作用域传递到独立作用域的原因，无论是直接传递还是使用表达式生成一个新值，像下面这样：

```js
it('evaluates isolate scope expression on parent scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        myAttr: '<'
      },
      link: function(scope) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    $rootScope.parentAttr = 41;
    var el = $('<div my-directive my-attr="parentAttr + 1"></div>');
    $compile(el)($rootScope);

    expect(givenScope.myAttr).toBe(42);
  });
});
```

让我们来看看怎么可以让这个单元测试通过。在做其它事情之前，我们需要对独立作用域绑定的解析器加入一些新的处理技巧：它需要能够处理属性绑定（@），也要能处理单向数据绑定（<）。下面这个正则表达式能够满足我们的需求：

```js
/\s*([@<])\s*(\w*)\s*/
```

其实也就是对之前的正则表达式进行扩展，现在可以接受`@`或`<`作为第一个字符，并把对其进行捕获，方便我们之后使用。

当我们在`parseIsolateBindings`应用这个正则表达式时，我们就可以为每一个绑定设置一个`mode`属性，以便之后使用：

_src/compile.js_

```js
function parseIsolateBindings(scope) {
  // var bindings = {};
  // _.forEach(scope, function(defnition, scopeName) {
    var match = defnition.match(/\s*([@<])\s*(\w*)\s*/);
    // bindings[scopeName] = {
      mode: match[1],
      attrName: match[2] || scopeName
  //   };
  // });
  // return bindings;
}
```

注意这里因为新增了一个捕获组，所以捕获组的获取索引也变了。

现在，我们需要在链接节点并创建独立作用域时处理`<`绑定模式。我们需要做的是：

1. 获取这个绑定对应的 DOM 上的表达式字符串
2. 把这个字符串当做 Angular 表达式来进行解析
3. 使用父作用来对解析后的表达式进行 evaluate
4. 把 evaluate 后的结果作为独立作用域上的属性值

下面就是上面4个步骤在代码中的呈现：

```js
_.forEach(
// newIsolateScopeDirective.$$isolateBindings,
// function(defnition, scopeName) {
//   var attrName = defnition.attrName;
//   switch (defnition.mode) {
//     case '@':
//       attrs.$observe(attrName, function(newAttrValue) {
//         isolateScope[scopeName] = newAttrValue;
//       });
//       if (attrs[attrName]) {
//         isolateScope[scopeName] = attrs[attrName];
//       }
//       break;
    case '<':
      var parentGet = $parse(attrs[attrName]);
      isolateScope[scopeName] = parentGet(scope);
      break;
  // }
// });
```

我们要使用之前实现了的`$parse`服务来对表达式进行解析，但在此之前，我们需要在`CompileProvider`的`$get`函数中注入这个服务：

```js
this.$get = ['$injector', '$parse', '$rootScope',
  function($injector, $parse, $rootScope) {

  // ...
  
  };
```
