### 指令优先级（Prioritizing Directives）

当对同一个元素应用多个指令时，应用的先后顺序不同往往会产生巨大的差异。一个指令很可能会依赖另一个指令，需要另一个指令先被应用上。

Angular 并不会把调整指令应用顺序的任务都放到应用开发者身上，指令系统会有一套内部的优先级配置系统对应用顺序进行控制。每个指令定义对象都可以设置一个`priority`属性，在指令被编译之前就回按照这个属性指定的顺序进行排列。优先级将会使用数字来表示，数字越大，优先级越高。也就是说，在编译时，指令会以降序排列。

而呈现在测试用例上，我们会定义两个指定了优先级的指令，然后看看他们是否会按预想的顺序进行编译：

_test/compile\_spec.js_

```js
it('applies in priority order', function() {
  var compilations = [];
  var injector = makeInjectorWithDirectives({
    lowerDirective: function() {
      return {
        priority: 1,
        compile: function(element) {
          compilations.push('lower');
        }
      };
    },
    higherDirective: function() {
      return {
        priority: 2,
        compile: function(element) {
          compilations.push('higher');
        }
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div lower-directive higher-directive></div>');
    $compile(el);
    expect(compilations).toEqual(['higher', 'lower']);
  });
});
```

如果两个指令有相同的优先级属性，那么我们就会使用指令的名称作为排序依据，这样我们就可以保持指令应用顺序是稳定的、可预测的：

```js
it('applies in name order when priorities are the same', function() {
  var compilations = [];
  var injector = makeInjectorWithDirectives({
    firstDirective: function() {
      return {
        priority: 1,
        compile: function(element) {
          compilations.push('first');
        }
      };
    },
    secondDirective: function() {
      return {
        priority: 1,
        compile: function(element) {
          compilations.push('second');
        }
      };
    }
  });
  injector.invoke(function($compile) {
    var el = $('<div second-directive first-directive></div>');
    $compile(el);
    expect(compilations).toEqual(['first', 'second']);
  });
});
```

如果两个指令的优先级和名称都一样，我们希望使用它们的注册顺序作为排序依据：

```js
it('applies in registration order when names are the same', function() {
  var compilations = [];
  var myModule = window.angular.module('myModule', []);
  myModule.directive('aDirective', function() {
    return {
      priority: 1,
      compile: function(element) {
        compilations.push('first');
      }
    };
  });
  myModule.directive('aDirective', function() {
    return {
      priority: 1,
      compile: function(element) {
        compilations.push('second');
      }
    };
  });
  var injector = createInjector(['ng', 'myModule']);
  injector.invoke(function($compile) {
    var el = $('<div a-directive></div>');
    $compile(el);
    expect(compilations).toEqual(['first', 'second'])
  });
});
```

> 这个测试用例目前就可以通过了，但为了测试的完整性，我们还是会加上它。

一旦我们在`collectDirectives`中收集到作用于某个元素的所有指令，我们会进行排序，然后再把它返回给调用方。我们可以使用 JavaScript 数组内建的 sort 方法，并马上会加入一个自定义的比较函数：

_src/compile.js_

```js
function collectDirectives(node) {
  var directives = [];

  // ...

  directives.sort(byPriority);
  return directives;
}
```

比较函数`byPriority`接收两个指令作为参数：

```js
function byPriority(a, b) {

}
```

这个函数会首先比较两个指令的优先级，如果返回值是负数，则是第一个指令优先级高，若是正数，则是第二个指令的优先级更高：

```js
function byPriority(a, b) {
  return b.priority - a.priority;
}
```

优先级相同时，我们按照名称进行排序。我们使用`<`来对两个字符串进行词汇比较即可：

```js
function byPriority(a, b) {
  var diff = b.priority - a.priority;
  if (diff !== 0) {
    return diff;
  } else {
    return (a.name < b.name ? -1 : 1);
  }
}
```

要比较指令名称，我们首先要能访问到指令名称，目前我们还没有实现对指令名称的访问。我们需要指令注册时进行记录，也就是在`$compileProvider.directive`方法中：

```js
$provide.factory(name + 'Directive', ['$injector', function($injector) {
  // var factories = hasDirectives[name];
  // return _.map(factories, function(factory) {
  //   var directive = $injector.invoke(factory);
  //   directive.restrict = directive.restrict || 'EA';
    directive.name = directive.name || name;
  //   return directive;
  // });
}]);
```

对于在优先级和指令名称都相同的情况，我们会使用指令的注册顺序作为排序依据，以保证编译顺序是稳定的、可预测的（先注册的先被编译）：

```js
function byPriority(a, b) {
  var diff = b.priority - a.priority;
  if (diff !== 0) {
    return diff;
  } else {
    if (a.name !== b.name) {
      return (a.name < b.name ? -1 : 1);
    } else {
      return a.index - b.index;
    }
  }
}
```

我们暂时还没有`index`属性，我们可以在指令注册期间加入：

```js
$provide.factory(name + 'Directive', ['$injector', function($injector) {
  // var factories = hasDirectives[name];
  return _.map(factories, function(factory, i) {
    // var directive = $injector.invoke(factory);
    // directive.restrict = directive.restrict || 'EA';
    // directive.name = directive.name || name;
    directive.index = i;
//     return directive;
  });
}]);
```

我们编写指令时也不用每次都指定一个优先级。如果没有指定优先级，则优先级的默认值为`0`：

_test/complie\_spec.js_

```js
it('uses default priority when one not given', function() {
  var compilations = [];
  var myModule = window.angular.module('myModule', []);
  myModule.directive('firstDirective', function() {
    return {
      priority: 1,
      compile: function(element) {
        compilations.push('first');
      }
    };
  });
  myModule.directive('secondDirective', function() {
    return {
      compile: function(element) {
        compilations.push('second');
      }
    };
  });
  var injector = createInjector(['ng', 'myModule']);
  injector.invoke(function($compile) {
    var el = $('<div second-directive first-directive></div>');
    $compile(el);
    expect(compilations).toEqual(['first', 'second']);
  });
});
```

这个默认值我们也可以在指令注册期间设置。指令的优先级，要么是已经被定义的值，要么是0：

_src/compile.js_

```js
$provide.factory(name + 'Directive', ['$injector', function($injector) {
  // var factories = hasDirectives[name];
  // return _.map(factories, function(factory, i) {
  //   var directive = $injector.invoke(factory);
  //   directive.restrict = directive.restrict || 'EA';
    directive.priority = directive.priority || 0;
  //   directive.name = directive.name || name;
    // directive.index = i;
  //   return directive;
  // });
}]);
```



