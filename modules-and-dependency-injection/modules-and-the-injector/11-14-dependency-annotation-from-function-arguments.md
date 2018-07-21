### 推断式依赖注入（Dependency Annotation from Function Arguments）

最后一种依赖注解方式也许是最有趣的一种，我们将不会显式地给出依赖名称，如果被注入的函数没有使用前两种注解方式，Angular 就会尝试从函数本身查找依赖，具体来说，是从函数的形参中进行查找。

从简单的开始实现，我们先处理无参数的情况：

test/injector\_spec.js

```js
it('returns an empty array for a non-annotated 0-arg function', function() {
  var injector = createInjector([]);
  var fn = function() {};
  
  expect(injector.annotate(fn)).toEqual([]);
});
```

为了让测试通过，我们先默认返回一个空数组：

src/injector.js

```js
function annotate(fn) {
  if (_.isArray(fn)) {
    return fn.slice(0, fn.length - 1);
  } else if (fn.$inject) {
    return fn.$inject;
  } else {
    return [];
  }
}
```

现在我们要着手处理带参数的情况了：

test/injector\_spec.js

```js
it('returns annotations parsed from function args when not annotated', function() {
  var injector = createInjector([]);
  var fn = function(a, b) { };
  
  expect(injector.annotate(fn)).toEqual(['a', 'b']);
});
```

要实现这一“黑魔法”有两个关键的步骤，第一步是获取函数源码（字符串），第二部是利用正则表达式进行过滤提取。在 JavaScript 中，我们可以通过调用 toString 原型方法来获取函数源码。

```js
(function(a, b) { }).toString() // => "function (a, b) { }"
```

函数源码中必然包含形参列表，我们可以用以下正则表达式来进行提取：

src/injector.js

```js
var FN_ARGS = /^function\s*[^\(]*\(\s*([^\)]*)\)/m;
```

为了让我们对正则表达式的结构有更清晰的了解，我们下面是对其进行拆解：

| /^ |  |
| :--- | :--- |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |
|  |  |



