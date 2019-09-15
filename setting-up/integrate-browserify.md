## 集成 Browserify（Integrate Browserify）

之前的设置会让 Karme 对所有在 `src` 和 `test` 目录下找到的所有 JavaScript 文件进行加载，并执行里面的全部内容。这意味着文件中的全局变量和函数，比如`sayHello`，能被其他文件访问到。

这种文件组织方式会让我们难以对某一点进行跟踪，因为我们要定位代码的位置很困难。官方的 AngularJS 代码就是使用这种方式的，但因为官方项目是在 2009 年创建的，当时并没有合适的替代方案。幸运的是，我们现在才开始创建，而目前已经有替代方案了。

Node.js 遵循一种打包标准，这种标准被称作 [CommonJS](http://wiki.commonjs.org/wiki/CommonJS)，它规定一个文件就是一个模块。一个模块可以使用 `require` 引入其他模块的东西，同时我们可以在模块中使用 `module.exports` 来定义可供对外输出的东西。这种模块系统非常适合我们使用。但我们要编写的不是 Node.js 代码，而是客户端的 JavaScript。但也不是没有办法，我们可以使用一个名为 `Browserify` 的工具，这样就能在客户端代码中使用模块系统。它会对我们所有的文件进行处理，最终输出一个能在浏览器（比如测试用的 PhantomJS 浏览器）中运行的包。

```bash
npm install --save-dev browserify watchify karma-browserify
```

让我们来看看代码中怎么来使用模块系统。我们应该在 `hello.js` 中对 `sayHello` 函数进行输出：

_src/hello.js_

```js
module.exports = function sayHello() {
  return 'Hello, world!';
};
```

然后我们就可以在测试用例中使用 `require` 进行加载：

_test/hello_spec.js_

```js
var sayHello = require('../src/hello');

describe('Hello', function() {

  it('says hello', function() {
    expect(sayHello()).toBe('Hello, world!');
  }); 

});
```

在我们运行这段代码之前，我们需要先在测试的配置中加入对 Browserify 的支持。我们已经安装了 `karma-browserify` 插件了，现在我们需要在 Karma 的配置文件中启用它：

_karma.conf.js_

```js
module.exports = function(config) {
  config.set({
    frameworks: ['browserify', 'jasmine'],
    files: [
      'src/**/*.js',
      'test/**/*_spec.js'
    ],
    preprocessors: {
      'test/**/*.js': ['jshint', 'browserify'],
      'src/**/*.js': ['jshint', 'browserify']
    },
    browsers: ['PhantomJS'],
    browserify: {
      debug: true
    }
  })
}
```