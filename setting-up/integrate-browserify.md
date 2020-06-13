### 集成 Browserify（Integrate Browserify）

目前 Karma 会对在 `src` 和 `test` 目录下的所有 JavaScript 文件都进行加载并执行。这意味着文件中的全局变量和函数，比如`sayHello`，能被其他文件直接访问到。

但这种文件组织方式在某种程度上会让代码变得难以跟踪，因为很难知道代码来自哪里。AngularJS 的官方代码就是使用这种方式组织的，但这是因为官方项目是在 2009 年创建时并没有合适的处理方案。幸运的是，当我们开始开发时已经有替代方案了。

Node.js 遵循的打包标准是 [CommonJS](http://wiki.commonjs.org/wiki/CommonJS)，这个标准规定一个文件就是一个模块。一个模块可以使用 `require` 语法引入其他模块的东西，同时我们可以使用 `module.exports` 语法定义当前模块中可被其他模块访问的东西。这套模块系统对我们来说就很合用了。但我们要编写的不是 Node.js 代码，而是客户端的 JavaScript 代码。难道就用不上了吗？其实我们可以使用一个叫 `Browserify` 的工具，它让我们在客户端代码中也能使用模块系统。`Browserify`会对所有文件进行处理，最终输出一个能在浏览器（比如我们测试用到的 PhantomJS 浏览器）中运行的包。

```bash
npm install --save-dev browserify watchify karma-browserify
```

我们来看看该如何使用这个工具。首先，我们在 `hello.js` 文件中导出 `sayHello` 函数：

_src/hello.js_

```js
module.exports = function sayHello() {
  return 'Hello, world!';
};
```

然后我们在测试用例中就能 `require` 语法加载这个函数：

_test/hello_spec.js_

```js
var sayHello = require('../src/hello');

describe('Hello', function() {

  it('says hello', function() {
    expect(sayHello()).toBe('Hello, world!');
  }); 

});
```

在运行代码之前，我们需要先在测试配置中集成对 Browserify 的支持。由于我们已经安装了 `karma-browserify` 插件，现在直接启用它就可以了：

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

这样配置好了以后，以后每次执行代码之前都会先运行 `browserify` 预处理器，此时，Browserify 就会对模块的 import 和 export 进行处理了。

要注意的是，我们同时启用了 Browserify 的 debug 功能，这意味着 Browserify 会在处理代码的同时生成代码的[source map](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/)，这样我们能从错误信息中看到发生错误的代码来源于哪个文件的哪一行。这个功能在我们要定位测试失败原因时会显得特别重要。

现在试一下启动 Karma，然后故意加入一些会让测试用例失败的代码试试。如果是语法错误，终端会出现一个 JSHint 错误信息，而如果单纯是单元测试失败，则会出现一个测试失败的提示信息。这两个错误都能在 Karma 的输出中看到。

> 运行单元测试时，偶尔会看到一个“Some of your tests did a full page reload!”的信息，但实际上你根本没有在单元测试中主动发起重新刷新页面的操作。虽然这个问题对我们的开发不会产生实质性的影响，但还是会让人分心。
> 
> 出现这个问题的原因在于 Karma 在侦测到文件变化后过早地执行测试用例了。你可以通过调高 Browserify 的 bundleDelay 配置参数来解决，直接把这个配置参数加入到 `karma.conf.js` 的 `browserify` 部分就可以了。这个配置参数的默认值为 700，而笔者在电脑上设置 `bundleDelay: 2000` 就可以解决这个问题了。