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

这个正则表达式是一个常量，会被放置到 injector.js 的顶部。为了让我们对正则表达式的结构有更清晰的了解，我们下面对其进行拆解：

| /^ | 我们会将匹配检索的开始位置放到源码的开头 |
| :--- | :--- |
| function | 函数当然是从 function 关键字开始 |
| \s\* | function 关键字后允许有若干空格，也允许无空格（匿名函数时） |
| \[^\\(\]\* | 允许有函数名，也允许无函数名称（匿名函数时） |
| \\( | 左括弧 |
| \s\* | 括弧后到第一个参数之间允许有若干空格，也允许无空格（无参数时） |
| \( | 开始对参数列表进行捕获 |
| \[^\\)\]\* | 参数列表，允许无参数 |
| \) | 结束对参数列表的捕获 |
| \\) | 右括弧 |
| /m | 允许多行检索（因为参数列表允许写成多行的） |

现在我们可以在 annotate 使用这个正则表达式进行匹配，匹配结果的第二个数组元素就是我们所需要的参数列表。当然目前这个参数列表还只是一个字符串，我们根据逗号 “,” 分割一下，就可以得到真正的参数列表。对于无参数的函数（判断是否无参数可以利用 Function.length 属性），我们需要特殊处理一下：

src/injector.js

```js
function annotate(fn) {
  if (_.isArray(fn)) {
    return fn.slice(0, fn.length - 1);
  } else if (fn.$inject) {
    return fn.$inject;
  } else if (!fn.length) {
    return [];
  } else {
    var argDeclaration = fn.toString().match(FN_ARGS);
    return argDeclaration[1].split(',');
  }
}
```

现在，你可能会发现测试用例还是没有通过，这是因为在参数b的前面有一个空格，所以我们实际上提取到的参数是 “ b”。再回顾当前的代码，我们会发现我们过滤了参数列表以前的空格，却没有过滤参数列表里面的空格。为了去除这些空格，我们需要对匹配出来的每一个参数名称进行过滤。

下面的这个正则表达式可以获取去除前后空格的字符串，满足我们的要求：

src/injector.js

```js
var FN_ARG = /^\s*(\S+)\s*$/;
```

接下来，我们就可以利用这个正则表达式来获取“纯粹”的依赖名称了：

src/injector.js

```js
function annotate(fn) {
  if (_.isArray(fn)) {
    return fn.slice(0, fn.length - 1);
  } else if (fn.$inject) {
    return fn.$inject;
  } else if (!fn.length) {
    return [];
  } else {
    var argDeclaration = fn.toString().match(FN_ARGS);
    return _.map(argDeclaration[1].split(','), function(argName) {
      return argName.match(FN_ARG)[1];
    });
  }
}
```

好了，现在测试用例终于可以通过了。但如果参数列表中出现了注释，又该怎么办呢？

test/injector\_spec.js

```js
it('strips comments from argument lists when parsing', function() {
  var injector = createInjector([]);
  var fn = function(a, /*b,*/ c) { };

  expect(injector.annotate(fn)).toEqual(['a', 'c']);
});
```

> 对于参数列表中出现注释，不同浏览器的处理方法不同。当我们调用 Function.toString\(\) 时，有些浏览器会帮我们把注释去除掉，有些不会。本书使用的 PhantomJS , 其 Webkit 内核可能会帮我们清理掉注释，所以在这个环境下测试用例可能会直接通过，但对于 Chrome 来说就不一定了。

这需要我们在提取之前先把函数源码里面的注释代码段去掉。我们先尝试一下下面的正则表达式：

src/injector.js

```js
var STRIP_COMMENTS = /\/\*.*\*\//;
```

这个正则将会帮我们找出以 “/\*“ 为开头、“\*/”为结尾的注释，然后我们使用 String.replace 进行去除：

```js
function annotate(fn) {
  if (_.isArray(fn)) {
    return fn.slice(0, fn.length - 1);
  } else if (fn.$inject) {
    return fn.$inject;
  } else if (!fn.length) {
    return [];
  } else {
    var source = fn.toString().replace(STRIP_COMMENTS, '');
    var argDeclaration = source.match(FN_ARGS);
    return _.map(argDeclaration[1].split(','), function(argName) {
      return argName.match(FN_ARG)[1];
    });
  }
}
```

但如果有多个注释代码段又该如何呢：

test/injector\_spec.js

```js
it('strips several comments from argument lists when parsing', function() {
  var injector = createInjector([]);
  var fn = function(a, /*b,*/ c /*, d*/ ) {};
  expect(injector.annotate(fn)).toEqual(['a', 'c']);
});
```

像上面的测试用例，按照当前的匹配规则，由于正则默认采用贪婪模式，"/\*b,\* c /\*, d\*/"这一代码也会匹配到，所以连有效参数 c 都会被清除掉。为了解决这些问题，我们需要升级一下目前匹配注释代码段的正则：

src/injector.js

```js
var STRIP_COMMENTS = /\/\*.*?\*\//g;
```

由于参数列表可以多行定义，所以我们还得对另一种注释形式——“//” 进行过滤：

test/injector\_spec.js

```js
it('strips // comments from argument lists when parsing', function() {
  var injector = createInjector([]);
  var fn = function(a, //b,
                    c) { };

  expect(injector.annotate(fn)).toEqual(['a', 'c']);
});
```

二次升级在所难免：

src/injector.js

```js
var STRIP_COMMENTS = /(\/\/.*$)|(\/\*.*?\*\/)/mg;
```

现在我们终于可以对烦人的注释代码段说“拜拜”了！

针对参数列表的提取，我们还剩下一个问题要解决。Angular 允许在参数名称两侧加上下划线"\_"，在提取注解时又会自动去除两侧的下划线。这个机制可以满足一种特殊需求——允许我们将一个依赖值赋值给本地同名变量：

```js
var aVariable;
injector.invoke(function(_aVariable_) {
  aVariable = _aVariable_;
});
```

> _译者注_：如果有一个本地变量b，我们需要在注射器函数中使用依赖b对本地变量b 进行赋值，我们可以在函数中传入形参 “\_b\_”，过滤下划线后我们可以顺利地获取到依赖 b 对应的缓存值 bValue，形参 “\_b\_”在函数被调用时自然被赋值为 bValue，因为名称不相同，"b = \_b\_" 可以完成需求。相反，如果我们不使用这种特殊处理方式，“b = b”的操作将毫无作用

除了在参数名称两侧加下划线这种情况，下划线在单侧出现，或者在中间出现，都不会触发这个机制：

test/injector_spec.js

```js
it('strips surrounding underscores from argument names when parsing', function() {
  var injector = createInjector([]);
  var fn = function(a, _b_, c_, _d, an_argument) { };

  expect(injector.annotate(fn)).toEqual(['a', 'b', 'c_', '_d', 'an_argument']);
});
```

我们将会把对双侧下划线的这种情况也放到 FN_ARG 正则中处理：

```js
var FN_ARG = /^\s*(_?)(\S+?)\1\s*$/;
```

由于 FN_ARG 的捕获组发生了调整，我们的匹配结果也会发生变化，具体是从匹配结果的第二个数组元素变到了第三个：

src/injector.js

```js
function annotate(fn) {
  if (_.isArray(fn)) {
    return fn.slice(0, fn.length - 1);
  } else if (fn.$inject) {
    return fn.$inject;
  } else if (!fn.length) {
    return [];
  } else {
    var source = fn.toString().replace(STRIP_COMMENTS, '');
    var argDeclaration = source.match(FN_ARGS);
    return _.map(argDeclaration[1].split(','), function(argName) {
      return argName.match(FN_ARG)[2];
    });
  }
}
```