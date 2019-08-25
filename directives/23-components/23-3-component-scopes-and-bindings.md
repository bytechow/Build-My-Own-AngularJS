### 组件作用域与绑定数据（Component Scopes and Bindings）

像`restrict`一样，组件也帮应用开发者决定了作用域的定义：组件都会有独立作用域。这个事实毋庸置疑。

_test/compile_spec.js_

```js
it('has an isolate scope', function() {
  var componentScope;
  var injector = makeInjectorWithComponent('myComponent', {
    controller: function($scope) {
      componentScope = $scope;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    expect(componentScope).not.toBe($rootScope);
    expect(componentScope.$parent).toBe($rootScope);
    expect(Object.getPrototypeOf(componentScope)).not.toBe($rootScope);
  });
});
```

我们只需要在组件的指令工厂函数中定义即可：

_src/compile.js_

```js
function factory() {
  return {
    restrict: 'E',
    controller: options.controller,
    scope: {}
  }; 
}
```

组件也肯定可以有绑定数据。当前指令绑定数据的所有类型（属性类、单向绑定类、双向绑定类、表达式类），都适用于组件。只是我们并不是通过`scope`属性来定义绑定数据，而是通过一个叫`bindings`的属性。当然，这些绑定数据都会绑定到控制器上，而不是作用域上。

_test/compile_spec.js_

```js
it('may have bindings which are attached to controller', function() {
  var controllerInstance;
  var injector = makeInjectorWithComponent('myComponent', {
    bindings: {
      attr: '@',
      oneWay: '<',
      twoWay: '='
    },
    controller: function() {
      controllerInstance = this;
    }
  });
  injector.invoke(function($compile, $rootScope) {
    $rootScope.b = 42;
    $rootScope.c = 43;
    var el = $('<my-component attr="a", one-way="b", two-way="c"></my-component>');
    $compile(el)($rootScope);
    
    expect(controllerInstance.attr).toEqual('a');
    expect(controllerInstance.oneWay).toEqual(42);
    expect(controllerInstance.twoWay).toEqual(43);
  });
});
```

要实现这个效果，我们可以利用控制器章节实现的一个便捷方法，也就是我们可以把一个对象作为`bindToController`的值来定义控制器绑定数据。我们直接将传入的`bindings`属性放到`bindToController`那里就可以了：

_src/compile.js_

```js
function factory() {
  return {
    restrict: 'E',
    controller: options.controller,
    scope: {},
    bindToController: options.bindings || {}
  }; 
}
```

另一样我们可以添加到组件作用域上的就是定义组件控制器的别名。它的工作方式跟常规指令是一样的，都是通过`controllerAs`属性：

_test/compile_spec.js_

```js
it('may use a controller alias with controllerAs', function() {
  var componentScope;
  var controllerInstance;
  var injector = makeInjectorWithComponent('myComponent', {
    controller: function($scope) {
      componentScope = $scope;
      controllerInstance = this;
    },
    controllerAs: 'myComponentController'
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    expect(componentScope.myComponentController).toBe(controllerInstance);
  });
});
````

开发时，我们就直接把`controllerAs`属性传递给指令工厂函数即可。

_src/compile.js_

```js
function factory() {
  return {
    restrict: 'E',
    controller: options.controller,
    controllerAs: options.controllerAs,
    scope: {},
    bindToController: options.bindings || {}
  };
}
```

控制器的别称也可以作为`controller`属性的一部分，这个字符串对应一个已经存在的控制器：

_test/compile_spec.js_

```js
it('may use a controller alias with "controller as" syntax', function() {
  var componentScope;
  var controllerInstance;
  var injector = createInjector(['ng', function($controllerProvider,
                                                $compileProvider) {
    $controllerProvider.register('MyController', function($scope) {
      componentScope = $scope;
      controllerInstance = this;
    });
    $compileProvider.component('myComponent', {
      controller: 'MyController as myComponentController'
    });
  }]);
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component');
    $compile(el)($rootScope);
    expect(componentScope.myComponentController).toBe(controllerInstance);
  });
});
```

我们无需对代码进行修改就可以完成这个功能，因为组件本身就是在指令的基础上构建出来的，而指令本身已经对这个功能进行了支持。

但组件控制器的别称跟普通指令还是有一点不同之处，那就是组件控制器是一直都有别称的。如果你没有定义别称，那组件控制器的别称就会默认设置为`$ctrl`。

_test/compile_spec.js_

```js
it('has a default controller alias of $ctrl', function() {
  var componentScope;
  var controllerInstance;
  var injector = makeInjectorWithComponent('myComponent', {
    controller: function($scope) {
      componentScope = $scope;
      controllerInstance = this;
    },
  });
  injector.invoke(function($compile, $rootScope) {
    var el = $('<my-component></my-component>');
    $compile(el)($rootScope);
    expect(componentScope.$ctrl).toBe(controllerInstance);
  });
});
```

我们可以在指令工厂函数中指定一个默认值：

_src/compile.js_

```js
function factory() {
  return {
    restrict: 'E',
    controller: options.controller,
    controllerAs: options.controllerAs || '$ctrl',
    scope: {},
    bindToController: options.bindings || {}
  };
}
```

这样处理后虽然能让这个单元测试通过，但却对前一个测试产生了影响。使用`controller: 'myController as ctrl'`的方式定义控制器别称已经行不通了！

原因是当我们在`$controller`服务对这类别称定义进行检查时出问题了，因为我们目前已经把`$ctrl`作为`controllerAs`的值，并且优先级更高。如果`controller`属性值中不包含别沉定义，我们就应该只用`$ctrl`。

那我们怎么才能知道属性值中有没有包含别称呢？我们在`controller.js`文件中已经有对这种情况进行验证的代码，但我们暂时还无法在`compile.js`中使用。我们需要来改一下。

首先，我们应该从`$controller`函数中抽离这个检测用的正则表达式到`controller.js`的顶层作用域，这样我们就不用再写一次了：

_src/controller.js_

```js
var CNTRL_REG = /^(\S+)(\s+as\s+(\w+))?/;
```

在`$controller`函数里，我们现在就直接用这个常量就可以了：

```js
var match = ctrl.match(CNTRL_REG);
```

然后，我们会在`controller.js`文件中定义一个函数，这个函数也会用到这个正则表达式。在这个函数中，如果找到对控制器别称的定义，那我们就把这个别称提取出来并返回。

```js
function identifierForController(ctrl) {
  if (_.isString(ctrl)) {
    var match = CNTRL_REG.exec(ctrl);
    if (match) {
      return match[3];
    }
  } 
}
```

现在，我们需要在`controller.js`文件中把这个函数和`$ControllerProvider`一起对外输出即可：

```js
module.exports = {
  $ControllerProvider: $ControllerProvider,
  identifierForController: identifierForController
};
```

由于我们对输出方式进行了修改，我们需要回到`angular_public.js`文件改变获取`$ControllerProvider`的方式。因为`$ControllerProvider`不再是`controller.js`的默认输出项，而是包含于一个对象中：

_src/angular_public.js_

```js
ngModule.provider('$controller', require('./controller').$ControllerProvider);
```

现在终于能够享受我们以上一翻操作的成果，我们可以在`compile.js`引入`identifierForController`函数，并把存放到顶层作用域的一个变量中。

_src/compile.js_

```js
var identifierForController = require('./controller').identifierForController;
```

我们会在用户没有传入`controllerAs`属性值时，尝试调用这个函数，若有返回值则该返回值作为组件控制器的别称，若没有返回值，我们再使用默认的`$ctrl`。

```js
function factory() {
  return {
    restrict: 'E',
    controller: options.controller,
    controllerAs: options.controllerAs ||
                  identifierForController(options.controller) ||
                  '$ctrl',
    scope: {},
    bindToController: options.bindings || {}
  };
}
```

这样就能令`controller as`的定义方式回复正常。现在，所有可能出现的控制器别名定义方式，我们都进行了支持。