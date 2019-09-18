### 集成 Browserify（Integrate Browserify）

目前我们的处理是，Karma 会对所有在 `src` 和 `test` 目录下的 JavaScript 文件都进行加载，并执行里面的全部内容。这意味着文件中的全局变量和函数，比如`sayHello`，能被其他文件访问到。

但这种文件组织方式在某种程度上会让代码难以跟踪，因为这样难以定位目标代码的位置。官方的 AngularJS 代码就是使用这种方式的，但这是因为官方项目是在 2009 年创建的，当时并没有真正合适的替代方案。幸运的是，目前已经有替代方案了。

Node.js 遵循的打包标准是 [CommonJS](http://wiki.commonjs.org/wiki/CommonJS)，这个标准规定一个文件就是一个模块。一个模块可以使用 `require` 语法引入其他模块的东西，同时我们可以在模块中使用 `module.exports` 语法定义模块中可对外输出的东西。这套模块系统就非常适合用来做文件组织了。但我们要编写的不是 Node.js 代码，而是客户端的 JavaScript 代码。不过也不是没有办法，我们可以使用一个名为 `Browserify` 的工具，它能使我们在客户端代码中使用模块系统。`Browserify`会对所有文件进行处理，最终输出一个能在浏览器（比如我们测试用到的 PhantomJS 浏览器）中运行的包。

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

有了这个配置文件，在执行代码之前，会通过了 `browserify` 进行预处理。此时，Browserify 会处理模块的 import 和 export。

注意，我们也启用了 Browserify 的 debug 功能，这意味着 Browserify 会在处理代码的同时输出 [source map](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/)，这样测试出现错误时我们能看到出错出现在哪个文件的哪一行中。如果测试出错，我们要定位错误原因时，这个特性是十分关键的。

现在试一下启动 Karma，然后加入一些会引起程序出错的代码看看效果。语法错误会导致一个 JSHint 错误信息的出现，而单元测试失败则会导致一个测试错误信息的出现。这两个错误都能在 Karma 的输出中看到。

> 当要运行单元测试的时候，偶尔会看到一个“Some of your tests did a full page reload!”的信息，即使当前你并没有在单元测试中做这样的事情。虽然并无大碍，但确实会让人分心。
> 
> 出现这个问题的原因是，在侦测到文件变化后，测试用例过早执行了。你可以通过调高 Browserify 的 bundleDelay 配置参数来解决，直接把这个配置参数加入到 `karma.conf.js` 的 `browserify` 部分就可以了。这个配置参数的默认值为 700，而在我的电脑上设置 `bundleDelay: 2000` 就可以解决这个问题了。