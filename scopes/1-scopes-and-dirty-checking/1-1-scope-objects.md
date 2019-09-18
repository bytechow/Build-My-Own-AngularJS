### 作用域对象（Scope Objects）

作用域对象可以通过对`Scope`构造函数应用`new`运算符进行调用的方式创建。运算结果就是一个普通的 JavaScript 对象。下面我们就来对这个基本行为创建第一个测试用例。

我们先创建专门用于测试 Angular 作用域的测试文件 `test/scope_spec.js`，然后往文件中加入以下内容：

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

在该文件的顶部我们启用了 [ES5 严格模式](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode)，并引入了 `Scope` 构造函数，这个构造函数我们希望从 `src` 目录的对应文件加载过来。在这个测试中，我们创建了一个 `Scope` 对象并对它赋值了一个属性，最后验证属性是否成功地添加到对象上。

> 加入了这个单元测试之后，如果此时你的终端正运行着 Karma，你应该能看到 Karma 报出了一个错误，这是因为我们还没有开发 `Scope` 构造函数。但这就是我们需要的效果，因为在测试驱动式开发中，看到自己设置的单元测试无法通过是很重要的一个步骤。
>
> 在本书的后续部分，我会假定你们在终端一直运行着 Karma，单元测试是不断被执行的，以后再有测试需要运行时就不再特别提醒了。

我们很轻易就可以让这个单元测试通过了：只需要创建 `src/scope.js` 文件，然后加入以下内容：

_src/scope.js_

```js
'use strict';

function Scope() {

}

module.exports = Scope;
```

在上面的单元测试中，我们在 scope 对象变量上添加了一个属性（叫做`aProperty`）。实际上这就是作用域属性的本质。它们都只是普通 JavaScript 对象上的属性，并没有什么特别之处。你不需要调用特殊的 setter 来生成它们，赋予它们的值也没有任何限制。而真正的“魔术”发生在两个非常特殊的函数——`$watch` 和 `$digest` 上。下面我们会把注意放到这两个函数上。