### 行内式依赖注入（Array-Style Dependency Annotation）

你可能不太喜欢使用 $inject 这么啰嗦的注解方式，幸亏 Angular 提供了一种更简洁的方式。我们可以用数组代替函数进行注解，在这个数组中，除了最后一个元素是被注入的函数，前面的数组元素都代表了一个依赖名：

```js
['a', 'b', function(one, two) {
  return one + two;
}]
```

由于我们允许采用多种注解方式，我们需要抽离出一个专门用于提取注解的方法。事实上，Angular中就在注射器中实现了这个方法，方法名就是 annotate。

我们需要在 describe 中对这个方法的行为进行定义。显然，当我们使用声明式注解方式，我们可以直接返回被注入函数的 $inject 属性作为 annotate 的返回值：

test/injector\_spec.js

```js
describe('annotate', function() {
  it('returns the $inject annotation of a function when it has one', function() {
    var injector = createInjector([]);
    var fn = function() {};
    fn.$inject = ['a', 'b'];

    expect(injector.annotate(fn)).toEqual(['a', 'b']);
  });
});
```

为此，我们需要在 createInjector 中新建一个内部函数，并对外暴露出该方法接口：

src/injector.js

```js
function createInjector(modulesToLoad) {
  var cache = {};
  var loadedModules = {};
  var $provide = {
    constant: function(key, value) {
      if (key === 'hasOwnProperty') {
        throw 'hasOwnProperty is not a valid constant name!';
      }
      cache[key] = value;
    }
  };

  function annotate(fn) {
    return fn.$inject;
  }

  function invoke(fn, self, locals) {
    var args = _.map(fn.$inject, function(token) {
      if (_.isString(token)) {
        return locals && locals.hasOwnProperty(token) ?
          locals[token] :
          cache[token];
      } else {
        throw 'Incorrect injection token! Expected a string, got ' + token;
      }
    });
    return fn.apply(self, args);
  }

  _.forEach(modulesToLoad, function loadModule(moduleName) {
    if (!loadedModules.hasOwnProperty(moduleName)) {
      loadedModules[moduleName] = true;
      var module = window.angular.module(moduleName);
      _.forEach(module.requires, loadModule);
      _.forEach(module._invokeQueue, function(invokeArgs) {
        var method = invokeArgs[0];
        var args = invokeArgs[1];
        $provide[method].apply($provide, args);
      });
    }
  });

  return {
    has: function(key) {
      return cache.hasOwnProperty(key);
    },
    get: function(key) {
      return cache[key];
    },
    annotate: annotate,
    invoke: invoke
  };
}
```

如果我们采用行内式依赖注解方式，我们就需要从数组中提取依赖名称数组：

test/injector\_spec.js

```js
it('returns the array-style annotations of a function', function() {
  var injector = createInjector([]);
  var fn = ['a', 'b', function() {}];

  expect(injector.annotate(fn)).toEqual(['a', 'b']);
});
```

所以当传入的参数是一个数组，我们就截取去除了最后一个元素的数组作为 annotate 的返回值：

src/injector.js

```js
function annotate(fn) {
  if (_.isArray(fn)) {
    return fn.slice(0, fn.length - 1);
  } else {
    return fn.$inject;
  }
}
```



