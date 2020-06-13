### 使用 Jasmine，Sinon 和 Karma 构建单元测试（Enable Unit Testing With Jasmine, Sinon, and Karma）

单元测试在我们的整个开发过程中都处于绝对的中心地位。这意味着我们需要一个好的测试框架。我们将要使用的测试框架是 [Jasmine](https://jasmine.github.io/2.3/introduction.html)，它有良好、简洁的 API，同时也满足我们所有的测试需求：

```js
describe('you can group test cases in "describe" blocks...', function() {

  describe('...which can also be nested', function() {

    it('test cases are in "it" blocks', function() {
      var string = 'where we can run arbitrary JavaScript code...';
      // ...and make assertions about results using "expect":
      expect(string).toEqual('expected string');
    });

  });

});
```

要真正运行单元测试，我们需要用到一个很受欢迎的测试执行器—— [Karma](http://karma-runner.github.io/)。通过安装特定插件后，Karma 和 Jasmine、Browserify 就能很好地被集成到一起使用了。

另外，我们还会用到一个测试辅助库—— [Sinon.JS](https://sinonjs.org/)，这个库能为我们提供一些之后会用到的较为复杂的 mock 对象。当我们开始开发 HTTP 服务时，Sinon 会变得非常有用。

我们先来安装 Jasmine 和 Sinon：

```bash
npm install --save-dev jasmine-core sinon
```

然后，我们再安装 Karma 和一些我们需要用到的 Karma 插件：

```bash
npm install --save-dev karma karma-jasmine karma-jshint-preprocessor
```

还要安装一个 PhantomJS 影子浏览器，Karma 实际上会在这个隐形的浏览器中运行（我们编写的）单元测试：

```bash
npm install --save-dev phantomjs-prebuilt@2.1.7 karma-phantomjs-launcher
```

> 译者注：原文的安装命令为 `npm install --save-dev phantomjs-prebuilt karma-phantomjs-launcher`。karma-phantomjs-launcher 用到需要到 2.1.7 版本的 phantomjs-prebuilt，所以这里给`phantomjs-prebuilt`加上了版本号。

下一步，我们会在 Karma 指定的配置文件 `karma.conf.js` 中加载并配置 Karma 和 Jasmine：

_karma.conf.js_

```js
module.exports = function(config) {
  config.set({
    frameworks: ['jasmine'],
    files: [
      'src/**/*.js',
      'test/**/*_spec.js'
    ],
    preprocessors: {
      'test/**/*.js': ['jshint'],
      'src/**/*.js': ['jshint']
    },
    browsers: ['PhantomJS']
  })
}
```

这样 Karma 就会对 `src` 目录下的所有 JavaScript 文件进行测试，而具体的测试用例会存放在 `test` 目录下对应的 JavaScript 文件中。Karma 运行时会持续监测这些文件，如果它们发生了改变，就重新运行测试套件。

我们还设置了一个 `jshint` 预处理器，在每个测试套件执行之前，我们都会先运行 JSHint 进行检查。

在测试代码中，我们会用到很多 Jasmine 自定义的全局变量，我们需要把这些变量都放到 JSHint 的 `globals` 对象中，以免 JSHint 报错：

_.jshintrc_

```json
{
  "browser": true,
  "browserify": true,
  "devel": true,
  "globals": {
    "jasmine": false,
    "describe": false,
    "it": false,
    "expect": false,
    "beforeEach": false,
    "afterEach": false
  }
}
```

我们同样可以使用 NPM 运行脚本简化 Karma 的启动命令。但在此之前，我们要让 `lint` 命令脚本中加入对 `test` 目录下的文件的 jshint 检查：

_package.json_

```json
"scripts": {
  "lint": "jshint src test",
  "test": "karma start"
}
```

这样我们就做好了运行单元测试的准备了。下面，我们先来创建一个测试文件，在 `test` 目录中新建 `hello_spec.js` 文件，然后往里面添加以下内容：

_test/hello_spec.js_

```js
describe('Hello', function() {

  it('says hello', function() {
    expect(sayHello()).toBe('Hello, world!');
  });

});
```

如果你曾经用过 Ruby 的 [rSpec](http://rspec.info/) ，应该不会对这种语法感到陌生：顶层作用域有一个 `describe` 代码块，代码块里可以存放了多个单元测试。而单个的单元测试是通过 `it` 代码块进行定义的，每个单元测试都定义了一个名称和一个测试函数。

> Jasmine （或 rSpec）使用的语法来源于 [行为驱动型开发模式](http://en.wikipedia.org/wiki/Behavior-driven_development)。`describe` 和 `it` 代码块一同定义了代码的行为。这也是我们为测试文件加上 `_spec.js` 后缀的原因，这些测试文件相当于代码的_说明书_。

下面我们启动 Karma 来执行这个单元测试：

```bash
npm test
```

这行命令会启动 Karma，然后在一个隐形的 PhantomJS 浏览器中进行测试。在 Karma 处于运行状态时，我们可以尝试更改 `src/hello.js` 和 `test/hello_spec.js` 的内容，看 Karma 是否能检测到文件发生了变化并重新运行测试套件。

在阅读本书的时候，我建议你一直运行着 Karma。这样你就不用一直通过手动的方式执行这些测试，如果代码出错，我们也能及时发现。

现在，我们的第一个单元已经测试通过了，可以继续下一步了。

