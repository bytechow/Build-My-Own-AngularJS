### 使用 watch 委托来优化 interpolation watch（Optimizing Interpolation Watches With A Watch Delegate）

interpolation 函数最终会被经常调用，因为`$compile`为它们设置的 watcher。另外，常见的应用中也会包含有大量的 interpolation，由于它们使用起来比较方便，在视图中被经常用到。

由于上面这些原因，我们有必要花一点时间来研究怎么优化 interpolation。Angular 在这里用了一个特有的优化手段，具体是通过减少 interpolation 结果的生成次数。优化要使用到的是 watch 委托——我们在本书第二部分介绍的一个特性。

interpolation 目前进行 watch 的对象是字符串里的表达式和静态文本部分都进行拼接后的结果。从概念上，这是我们想要的：当 DOM 中的文本值进行改变时更新 DOM。

但究竟到什么时候文本才发生改变呢？答案是它只会在至少一个表达式的值发生改变的时候进行改变。目前我们的做法是，每次 watch 运行的时候都生成一个新的字符串，无论这个字符串的内容是不是跟之前是一致的，这会导致我们做很多无用功，而且也会给垃圾收集系统带来压力。如果我们能够做到——所有表达式的值都没有改变，就不再新建一个字符串，那我们就能提高运行效率了。这里使用 watch 委托就能起到一个很好的效果。

再回顾一下，watch 委托是指可以作为`$$watchDelegate`属性的值加入到 watch 函数中的一个特殊方法。如果我们加入了这个属性，scope 中的 watcher 将会使用这个委托来对值变更进行检测，而不再适用 watch 函数本身的返回值。

针对这种优化需求，我们可以构建一个 watch delegate，其中会对表达式进行检测，如果没有表达式的结果值发生变化，就不生成新的结果字符串了。

作为开头，我们需要测试一下`$interpolate`返回的 interpolation 函数，是否真的有 watch 委托：

_test/interpolate\_spec.js_

```js
it('uses a watch delegate', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');
  var interp = $interpolate('has an {{expr}}');
  expect(interp.$$watchDelegate).toBeDefined();
});
```

我们可以利用之前为 interpolation 函数建立的对象扩展来增加这个属性。回顾 watch 委托，它会接收两个参数，一个是完成 watch 时所在的那个 scope，还有一个在监测到变化就调用的 listener 函数：

_src/interpolate.js_

```js
return _.extend(function interpolationFn(context) {
  // return _.reduce(parts, function(result, part) {
  //   if (_.isFunction(part)) {
  //     return result + stringify(part(context));
  //   } else {
  //     return result + part;
  //   }
  // }, '');
}, {
  // expressions: expressions,
  $$watchDelegate: function(scope, listener) {
    
  }
});
```

这样我们最近建立的那个单元测试就能通过了，但却也影响了之前的一些个测试。那是因为 scope 现在采用了`$$watchDelegate`，但这个委托现在还没干任何事情。我们将使用目前存在的单元测试作为指南，告诉我们什么时候会有 watch delegate 在使用：当它们都重新通过，我们就完成了这个问题的修复了。

在 watch 委托中，我们需要监测 interpolation 字符串中的任何一个表达式，看它们的值是否发生了变化。现在我们还没有解析后的表达式函数的集合，所以我们得先加上：

_src/interpolate.js_

```js
function $interpolate(text, mustHaveExpressions) {
  // var index = 0;
  // var parts = [];
  // var expressions = [];
  var expressionFns = [];
  // var startIndex, endIndex, exp, expFn;
  // while (index < text.length) {
  //   startIndex = text.indexOf('{{', index);
  //   if (startIndex !== -1) {
  //     endIndex = text.indexOf('}}', startIndex + 2);
  //   }
  //   if (startIndex !== -1 && endIndex !== -1) {
  //     if (startIndex !== index) {
  //       parts.push(unescapeText(text.substring(index, startIndex)));
  //     }
  //     exp = text.substring(startIndex + 2, endIndex);
  //     expFn = $parse(exp);
  //     parts.push(expFn);
  //     expressions.push(exp);
      expressionFns.push(expFn);
  //     index = endIndex + 2;
  //   } else {
  //     parts.push(unescapeText(text.substring(index)));
  //     break;
  //   }
  // }

  // ...

}
```

现在，我们就可以对这些表达式函数进行 watch 了。我们可以利用 Scope 的 API `$watchGroup` 来一次性对它们全部进行 watch：

```js
$$watchDelegate: function(scope, listener) {
  return scope.$watchGroup(expressionFns, function() {
    
  });
}
```

我们需要在 watch group 的 listener 函数中做的是构造结果字符串，并传给 listener 函数。我们需要在里面做跟 interpolation 函数内部一样的 reduce 操作来拼合字符串：

```js
$$watchDelegate: function(scope, listener) {
  return scope.$watchGroup(expressionFns, function() {
    listener(_.reduce(parts, function(result, part) {
      if (_.isFunction(part)) {
        return result + stringify(part(scope));
      } else {
        return result + part;
      }
    }, ''));
  });
}
```

由于这里重复了，我们可以引入一个叫`compute`的函数来封装 reduce 的过程，只需要给这个函数传递一个上下文对象即可：

```js
function compute(context) {
  return _.reduce(parts, function(result, part) {
    if (_.isFunction(part)) {
      return result + stringify(part(context));
} else {
      return result + part;
    }
}, ''); }
```

然后，我们就可以在 interpolation 函数和 watch 委托直接使用`compute`函数即可，无需重复相同的代码：

```js
return _.extend(function interpolationFn(context) {
  return compute(context);
}, {
  expressions: expressions,
  $$watchDelegate: function(scope, listener) {
    return scope.$watchGroup(expressionFns, function() {
      listener(compute(scope));
    });
  }
});
```

这个 watch 委托目前还是比较随意的，因为它还没有满足 listener 函数调用的约定：listener 函数调用时需要同时传入新值和旧值（还有 scope 对象）。我们现在只是传入了新值，这不足够：

_test/interpolate_spec.js_

```js
it('correctly returns new and old value when watched', function() {
  var injector = createInjector(['ng']);
  var $interpolate = injector.get('$interpolate');
  var $rootScope = injector.get('$rootScope');

  var interp = $interpolate('{{expr}}');
  var listenerSpy = jasmine.createSpy();
  
  $rootScope.$watch(interp, listenerSpy);
  $rootScope.expr = 42;
  
  $rootScope.$apply();
  expect(listenerSpy.calls.mostRecent().args[0]).toEqual('42');
  expect(listenerSpy.calls.mostRecent().args[1]).toEqual('42');
  
  $rootScope.expr++;
  $rootScope.$apply();
  expect(listenerSpy.calls.mostRecent().args[0]).toEqual('43');
  expect(listenerSpy.calls.mostRecent().args[1]).toEqual('42');
});
```

在 watch 委托中，我们需要保存上一次的计算值，并把它作为旧值传入给 listener 函数进行调用。同时，我们也把 scope 也传递过去，这样就完整地满足约定了：

_src/interpolate.js_

```js
$$watchDelegate: function(scope, listener) {
  var lastValue;
  return scope.$watchGroup(expressionFns, function() {
    var newValue = compute(scope);
    listener(newValue, lastValue, scope);
    lastValue = newValue;
  });
}
```

listener 函数的约定还包括要求在 watch 第一次运行时，新值和旧值要保持一致。这条约定我们也需要遵守。如果 watch group 的新值和旧值是一样的话，传入 listener 函数的也会是相同的：

```js
$$watchDelegate: function(scope, listener) {
  var lastValue;
	return scope.$watchGroup(expressionFns, function(newValues, oldValues) {
		var newValue = compute(scope);
		listener(
			newValue,
			(newValues === oldValues ? newValue : lastValue),
			scope 
		);
    lastValue = newValue;
  });
}
```