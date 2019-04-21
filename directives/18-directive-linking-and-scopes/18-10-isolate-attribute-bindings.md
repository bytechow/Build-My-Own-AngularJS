### 独立属性绑定（Isolate Attribute Bindings）

现在我们有了独立作用域，但它们还是“白纸一片”。这样它们的作用还是比较有限，而就如我们之前讨论过的，我们可以有几种方法往独立作用域绑定数据。

我们要介绍的第一个方法是在元素属性上绑定作用域相关的属性值。这个作用域属性将会利用到我们上面章节实现的属性监视机制，这样每当元素属性通过`$set`来进行修改，我们都会对作用域属性进行更新。

属性绑定在以下几个方面显得比较有用：你可以很方便地对定义在 DOM 或 HTML 上的元素属性进行访问，你也可以通过设置这样的属性来进行独立作用域指令和其他指令的通信。这是因为元素上的所有指令，无论是独立作用域还是普通作用域，都共享同一个属性对象。

属性绑定会在指令定义对象的 scope 属性中进行定义。key 自然是用于定义属性的名称，而属性值就是字符`@`，这是代表"属性绑定"的缩写。一旦我们加入了这个属性定义，以后再对这个属性进行`$set`的时候，它就会通知独立作用域：

_test/compile_spec.js_

```js
it('allows observing attribute to the isolate scope', function() {
  var givenScope, givenAttrs;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        anAttr: '@'
      },
      link: function(scope, element, attrs) {
        givenScope = scope;
        givenAttrs = attrs;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive></div>');
    $compile(el)($rootScope);

    givenAttrs.$set('anAttr', '42');
    expect(givenScope.anAttr).toEqual('42');
  });
});
```

接下来进行开发。进行独立作用域绑定的第一步发生在指令注册期间。如果指令声明了独立作用域，我们会对它们的内容进行解析，以便下一步操作：

```js
$provide.factory(name + 'Directive', ['$injector', function($injector) {
  var factories = hasDirectives[name];
  return _.map(factories, function(factory, i) {
    var directive = $injector.invoke(factory);
    directive.restrict = directive.restrict || 'EA';
    directive.priority = directive.priority || 0;
    if (directive.link && !directive.compile) {
      directive.compile = _.constant(directive.link);
    }
    if (_.isObject(directive.scope)) {
      directive.$$isolateBindings = parseIsolateBindings(directive.scope);
    }
    directive.name = directive.name || name;
    directive.index = i;
    return directive;
  });
}]);
```

我们需要在`compile.js`文件的顶层作用域中加入新函数`parseIsolateBindings`。它接收作用域定义对象并返回一个对象，这个对象存放了对作用域对象进行解析后的结果。目前来说，我们先直接接收这个定义对象就好了。这个函数会接收一个作用域定义对象，例如：

```js
{
  anAttr: '@'
}
```

然后返回下面的结果就好：

```js
{
  anAttr: {
    mode: '@'
  }
}
```

之后我们会实现更丰富的功能，但目前我们只需要像下面这样实现即可：

_src/compile.js_

```js
function parseIsolateBindings(scope) {
  var bindings = {};
  _.forEach(scope, function(defnition, scopeName) {
    bindings[scopeName] = {
      mode: defnition
    };
  });
  return bindings;
}
```

处理独立作用域绑定的第二步，是在指令被链接时执行真正的绑定。这会发生在节点链接函数，我们会对之前创建的`$$isolateBindings`对象进行遍历：

```js
function nodeLinkFn(childLinkFn, scope, linkNode) {
  var $element = $(linkNode);

  var isolateScope;
  if (newIsolateScopeDirective) {
    isolateScope = scope.$new(true);
    $element.addClass('ng-isolate-scope');
    $element.data('$isolateScope', isolateScope);

    _.forEach(
      newIsolateScopeDirective.$$isolateBindings,
      function(defnition, scopeName) {

      }
    );
  }
  // ...
}
```

目前，我们只需要检测绑定属性的类型是不是代表属性绑定的`@`，如果是的话，对相应的这个元素属性添加一个监视器。而这个观察者会把属性值添加到作用域对象上。

```js
// _.forEach(
//   newIsolateScopeDirective.$$isolateBindings,
//   function(defnition, scopeName) {
    switch (defnition.mode) {
      case '@':
        attrs.$observe(scopeName, function(newAttrValue) {
          isolateScope[scopeName] = newAttrValue;
        });
        break;
//     }
//   }
// );
```

`$observe`只会在下一次属性发生变化时才会被调用，我们希望即使属性即使之后不发生变化也会在作用域上有它的值存在，也就是说我们需要马上把属性的初始值放到作用域上，这样我们就可以在链接函数运行之前准备好属性值：

_test/compile_spec.js_

```js
it('sets initial value of observed attr to the isolate scope', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        anAttr: '@'
      },
      link: function(scope, element, attrs) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive an-attr="42"></div>');
    $compile(el)($rootScope);
    expect(givenScope.anAttr).toEqual('42');
  });
});
```

我们可以在注册监视器做这个工作：

_src/compile.js_

```js
_.forEach(
  newIsolateScopeDirective.$$isolateBindings,
  function(definition, scopeName) {
    // switch (definition.mode) {
    //   case '@':
    //     attrs.$observe(scopeName, function(newAttrValue) {
    //       isolateScope[scopeName] = newAttrValue;
    //     });
        if (attrs[scopeName]) {
          isolateScope[scopeName] = attrs[scopeName];
        }
        // break;
    // }
  }
);
```

目前元素上的属性和独立作用域上的属性是一一对应的关系。但实际上，我们可以对作用域属性指定一个不同的名称。我们可以用作用域属性名作为作用域定义对象的 key，而将元素上对应的属性用`@`前缀加上它的驼峰形式进行指定：

_test/compile_spec.js_

```js
it('allows aliasing observed attribute', function() {
  var givenScope;
  var injector = makeInjectorWithDirectives('myDirective', function() {
    return {
      scope: {
        aScopeAttr: '@anAttr'
      },
      link: function(scope, element, attrs) {
        givenScope = scope;
      }
    };
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<div my-directive an-attr="42"></div>');
    $compile(el)($rootScope);
    expect(givenScope.aScopeAttr).toEqual('42');
  });
});
```

我们需要对作用域定义对象的值进行加入一些解析来获取作用域别名。因为有了依赖注入机制，我们可以很方便滴引入一个正则表达式。我们可以使用下面这个正则表达式，可以匹配一个`@`字符，下面可以有0个或多个字符，这些字符会被捕捉为一组。另外，这个表达式也支持在单词之间有空格存在：

```js
/\s*@\s*(\w*)\s*/
```

使用这个正则表达式捕捉到的值会变成绑定对象中的`attrName`属性值。因为这个别称是可选的，所以`attrName`也可以是作用域的属性名称，这也是我们之前处理过的情况：

_src/compile.js_

```js
function parseIsolateBindings(scope) {
  // var bindings = {};
  // _.forEach(scope, function(defnition, scopeName) {
    var match = defnition.match(/\s*@\s*(\w*)\s*/);
    // bindings[scopeName] = {
      mode: '@',
      attrName: match[1] || scopeName
  //   };
  // });
  // return bindings;
}
```

现在在设置独立作用域绑定时，我们必须使用`attrName`这个名称来对元素属性访问：

_src/compile.js_

```js
_.forEach(
  newIsolateScopeDirective.$$isolateBindings,
  function(defnition, scopeName) {
    var attrName = defnition.attrName;
    // switch (defnition.mode) {
    //   case '@':
        attrs.$observe(attrName, function(newAttrValue) {
        //   isolateScope[scopeName] = newAttrValue;
        });
        if (attrs[attrName]) {
    //       isolateScope[scopeName] = attrs[attrName];
        }
    //     break;
    // }
  }
);
```