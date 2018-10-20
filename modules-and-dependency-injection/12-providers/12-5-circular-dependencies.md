### 处理循环依赖问题（Circular Dependencies）

既然支持依赖注入其它依赖，我们就可能会遇到循环依赖问题。假设A依赖B，B依赖C，C又依赖A，一旦我们构建类似的依赖，程序就会陷入死循环，最终抛出堆栈溢出的错误。但是我们希望有一个更清晰的错误提示：

test/injector\_spec.js

```js
it('notifes the user about a circular dependency', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', {
    $get: function(b) {}
  });
  module.provider('b', {
    $get: function(c) {}
  });
  module.provider('c', {
    $get: function(a) {}
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.get('a');
  }).toThrowError(/Circular dependency found/);
});
```

要满足这一需求，有两个关键点。第一个关键点在于，当我们调用某个 provider 的 $get 方法生成依赖之前时，我们会在其对应的实例依赖中加入一个标识（marker），这个标识可以记录该 provider 依赖的状态为“正在生成中”，这样当下次我们再次访问这个依赖时，我们就知道发生了循环依赖。

我们可以采用一个空对象作为标识（marker）,因为空对象不等于除它自己以外的任何值：

src/injector.js

```js
var FN_ARGS = /^function\s*[^\(]*\(\s*([^\)]*)\)/m;
var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
var STRIP_COMMENTS = /(\/\/.*$)|(\/\*.*?\*\/)/mg;
var INSTANTIATING = { };
```

在 getService 函数中，我们会在调用 $provider 之前在对应的实例依赖放入标识，并在每次调用 getService 时先检查对应的实例依赖是否已经被标记，若已经被标记，则说明出现了循环依赖：

src/injector.js

```js
function getService(name) {
  if (instanceCache.hasOwnProperty(name)) {
    if (instanceCache[name] === INSTANTIATING) {
      throw new Error('Circular dependency found');
    }
    return instanceCache[name];
  } else if (providerCache.hasOwnProperty(name + 'Provider')) {
    instanceCache[name] = INSTANTIATING;
    var provider = providerCache[name + 'Provider'];
    var instance = instanceCache[name] = invoke(provider.$get);
    return instance;
  }
}
```

然后当依赖最终被创建完成时，自然会把 INSTANTIATING 标识替换为依赖值。但在实例化过程中也可能出错，如果出错，我们不希望这个实例化标识还留在实例依赖中：

test/injector\_spec.js

```js
it('cleans up the circular marker when instantiation fails', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', {
    $get: function() {
      throw 'Failing instantiation!';
    }
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.get('a');
  }).toThrow('Failing instantiation!');
  expect(function() {
    injector.get('a');
  }).toThrow('Failing instantiation!');
});
```

上面测试用例的做法是，尝试获取一个抛出特殊错误的 provider 依赖，并重复执行，看两次执行结果是不是都抛出了这个错误。现在测试用例不通过，原因是我们第一次获取时把实例化标识放到了 provider a 对应的实例依赖中，第二次调用的时候，由于没有对 provider a 的实例进行更新，当检查是否发生了循环依赖，发现实例已经被附上了实例化标记，程序认定出现了循环依赖，所以抛出的是循环依赖的错误。我们应该保证无论程序是否抛出错误，都要清除实例化标识：

src/injector.js

```js
function getService(name) {
  if (instanceCache.hasOwnProperty(name)) {
    if (instanceCache[name] === INSTANTIATING) {
      throw new Error('Circular dependency found');
    }
    return instanceCache[name];
  } else if (providerCache.hasOwnProperty(name + 'Provider')) {
    instanceCache[name] = INSTANTIATING;
    try {
      var provider = providerCache[name + 'Provider'];
      var instance = instanceCache[name] = invoke(provider.$get);
      return instance;
    }
    finally {
      if (instanceCache[name] === INSTANTIATING) {
        delete instanceCache[name];
      }
    }
  }
}
```

另外，只是通知用户出现了循环依赖还是没什么实际用处，我们应该提供一种更好的方式让用户知道究竟发生了什么。

> 在本书中，我们很少讲述错误处理，这里我们将要介绍其中一种组织错误提示信息的方法。

我们要做的是，把循环依赖的路径打印出来，注意，我们的依赖顺序是从右往左：

```
a <- c <- b <- a
```

test/injector\_spec.js

```js
it('notifes the user about a circular dependency', function() {
  var module = window.angular.module('myModule', []);
  module.provider('a', {
    $get: function(b) {}
  });
  module.provider('b', {
    $get: function(c) {}
  });
  module.provider('c', {
    $get: function(a) {}
  });
  var injector = createInjector(['myModule']);
  expect(function() {
    injector.get('a');
  }).toThrowError('Circular dependency found: a <- c <- b <- a');
});
```

要完成这一需求，我们在加载依赖的过程中保存当前的依赖路径，让我们在注射器中新增一个数组来进行记录：

src/injector.js

```js
function createInjector(modulesToLoad, strictDi) {
  var providerCache = {};
  var instanceCache = {};
  var loadedModules = {};
  var path = [];
  // ...
```

在 getService 函数中，我们会把 path 作为一个栈空间对依赖路径进行存储。当我们开始解决某个依赖，我们会把依赖的名字放到数组的开头。如果依赖解决完成（获取成功），我们就把它的名字从 path 中去除：

src/injector.js

```js
function getService(name) {
  if (instanceCache.hasOwnProperty(name)) {
    if (instanceCache[name] === INSTANTIATING) {
      throw new Error('Circular dependency found');
    }
    return instanceCache[name];
  } else if (providerCache.hasOwnProperty(name + 'Provider')) {
    path.unshift(name);
    instanceCache[name] = INSTANTIATING;
    try {
      var provider = providerCache[name + 'Provider'];
      var instance = instanceCache[name] = invoke(provider.$get);
      return instance;
    }
    finally {
      path.shift();
      if (instanceCache[name] === INSTANTIATING) {
        delete instanceCache[name];
      }
    }
  }
}
```

如果发生了循环依赖，我们在加入了当前发生循环依赖的依赖名之后，就可以把循环依赖发生的完整路径展示出用户看了：

src/injector.js

```js
function getService(name) {
  if (instanceCache.hasOwnProperty(name)) {
    if (instanceCache[name] === INSTANTIATING) {
      throw new Error('Circular dependency found: ' +
        name + ' <- ' + path.join(' <- '));
    }
    return instanceCache[name];
  } else if (providerCache.hasOwnProperty(name + 'Provider')) {
    path.unshift(name);
    instanceCache[name] = INSTANTIATING;
    try {
      var provider = providerCache[name + 'Provider'];
      var instance = instanceCache[name] = invoke(provider.$get, provider);
      return instance;
    }
    fnally {
      path.shift();
    }
  }
}
```

现在用户更清楚错误产生的原因是什么了！

