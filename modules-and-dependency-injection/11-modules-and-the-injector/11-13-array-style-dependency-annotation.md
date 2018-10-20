### 行内式依赖注入（Array-Style Dependency Annotation）

如果你觉得 $inject 注解的方式比较繁琐，Angular 还提供了一种更简洁的方式。我们可以用数组代替函数进行注解，在这个数组中，除了最后一个元素是注射函数，在此之前的数组元素就代表了要注入的依赖：

```js
['a', 'b', function(one, two) {
  return one + two;
}]
```

由于我们允许采用多种注解方式，我们需要提取出一个专门用于解析注解的方法，也就是 annotate。

我们需要在测试文件中对 annotate 的行为进行定义。首先，对于声明式注解方式，我们可以直接返回注射函数的 $inject 属性值作为 annotate 的返回值：

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

下面，我们先在 createInjector 中新建一个内部函数，并对外暴露出该方法接口：

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



