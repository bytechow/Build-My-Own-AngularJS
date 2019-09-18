### 安装 Lo-Dash 和 jQuery（Include Lo-Dash And jQuery）

官方的 Angular 项目本身并不依赖任何第三方库（但如果项目引入了 jQuery 库，Angular 会转而直接使用这个库）。但在我们的开发过程中，为了集中精力研究 Angular 框架的原理，将一些底层细节交给已有的第三方库来处理也无可厚非。

有两类底层操作我们会找第三方库来帮忙处理：

- 数组和对象的操作，比如遍历，相等性比较和克隆，我们会交给 Lo-Dash 来完成。
- DOM 查询与操作，我们会交给 jQuery 来完成。

这两个库都可以通过 NPM 包获取。下面我们来安装它们：

```bash
npm install --save lodash@4 jquery
``` 

我们使用与之前安装 Browserify 和 Karma 相同的方法来安装这两个库。不同之处在于我们这次使用的参数是 `--save` 而不是 `--save-dev`，这意味着这两个库不仅仅会在开发阶段用到，还会作为运行时的依赖。

我们可以对“Hello, world!”程序做一些改动来验证两个库是否可以正常使用。安装了 LoDash 的 NPM 包后，我们只需要直接`require`这个库就好了：

_src/hello.js_

```js
var _ = require('lodash');

module.exports = function sayHello(to){
  return _.template('Hello, <%= name %>!')({name: to})
}
```

我们将要打招呼的对象名称变成参数传入，然后利用 Lo-Dash 的 [template](https://lodash.com/docs#template) 函数生成结果字符串。

在单元测试中，我们也要进行对应的更改：

_test/hello_spec.js_

```js
var sayHello = require('../src/hello');

describe('Hello', function() {
  
  it('says hello', function() {
    expect(sayHello('Jane')).toBe('Hello, Jane!');
  });

});
```