### 作用域对象（Scope Objects）

我们可以通过使用 `new` 运算符来调用 `Scope()` 创建一个作用域对象，结果会返回一个普通的 JavaScript 对象。下面我们就来对这个基本行为创建第一个测试用例。

我们先来创建专门用于测试 Angular 作用域的测试文件 `test/scope_spec.js`，然后往文件中加入以下内容：

_test/scope_spec.js_

```js
'use strict';

var Scope = require('../src/scope');

describe('Scope', function() {

  it('can be constructed and used as an object', function() {
    var scope = new Scope();
    scope.aProperty = 1;
  
    expect(scope.aProperty).toBe(1);
  });

});
```

在这个测试文件的顶部我们启用了 [ES5 严格模式](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)，并引入了 `src` 目录下的 `Scope` 构造函数。在这个测试中，我们创建了一个 `Scope` 对象并给它加上了一个属性，最后验证属性是否成功地添加到这个对象上。

> 如果此时你的终端正运行着 Karma，在加入了这个单元测试之后，你会能看到 Karma（在终端）报出了一个错误，原因是我们还没有实现 `Scope` 构造函数。这恰恰就是我们需要的效果，因为在“测试驱动型”的开发过程中，看到自己设置的单元测试无法通过也是重要的一步。
>
> 在本书后面的讲解中，我会假定你的终端一直运行着 Karma，单元测试会被持续执行的，不会再特别提醒你要运行 Karma 进行测试了。

只需要创建 `src/scope.js` 文件，然后加入以下内容，就能轻松地通过我们的第一个测试用例了：

_src/scope.js_

```js
'use strict';

function Scope() {

}

module.exports = Scope;
```

在上面的单元测试中，我们给 scope 对象添加了一个属性（叫做`aProperty`）。实际上，作用域属性的本质就是 JavaScript 对象上的属性，并没有什么特别之处。你不需要调用特殊的 setter 来生成它们，赋值内容也没有限制。真正神奇的是两个非常特殊的函数——`$watch` 和 `$digest` 上。下面我们来看看这两个函数到底有什么特别。