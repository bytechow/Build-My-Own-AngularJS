### 使用 Jasmine，Sinon 和 Karma 构建单元测试（Enable Unit Testing With Jasmine, Sinon, and Karma）

单元测试在整个开发过程中都处于绝对的中心地位。这意味着我们需要一个好的测试框架。我们将要使用的测试框架是 [Jasmine](https://jasmine.github.io/2.3/introduction.html)，它有良好、简洁的 API，同时也满足我们所有的测试需求：

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

要真正运行单元测试，我们需要用到一个很受欢迎的测试执行器—— [Karma](http://karma-runner.github.io/)。安装插件后，它就能与 Jasmine 和 Browserify 良好地集成到在一起使用。

另外，我们还会用到一个测试助手库—— [Sinon.JS](https://sinonjs.org/)，这个库能为我们提供之后会用到的一些较为复杂的 mock 对象。在我们开始构建 HTTP 服务时，Sinon 会变得非常有用。

我们先来安装 Jasmine 和 Sinon：

```bash
npm install --save-dev jasmine-core sinon
```

然后，我们再安装 Karma 和一些我们需要用到的 Karma 插件：

```bash
npm install --save-dev karma karma-jasmine karma-jshint-preprocessor
```

还要安装一个 PhantomJS 影子浏览器，Karma 会在这个隐形的浏览器中运行单元测试：

```bash
npm install --save-dev phantomjs-prebuilt karma-phantomjs-launcher
```

下一步，我们会在 Karme 特有的配置文件 `karma.conf.js` 加载并配置 Karme 和 Jasmine：

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

这样，Karma 就会对 `src` 目录下的所有 JavaScript 文件进行测试。测试本身则存放在 `test` 目录下。当 Karma 运行时，会一直监测 `src` 目录，看它里面的文件是否发生了改变，如果发生了变化则自动重新运行测试套件。

这里，我们还会设置一个 `jshint` 预处理器，在每个测试套件启动之前，我们都会先运行 JSHint。

在单元测试中，我们会用到很多由 Jasmine 定义的全局变量，因此需要把我们要用到的 Jasmine 全局变量都放到 JSHint 的 `globals` 对象中，以免 JSHint 报错：

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

我们也可以通过添加一个 NPM 脚本来简化 Karma 的启动命令。在此之前，我们先加入`lint` 运行脚本，执行这个脚本时，会对 `test` 目录下的进行 jshint 检查：

_package.json_

```json
"scripts": {
  "lint": "jshint src test",
  "test": "karma start"
}
```

现在我们可以运行单元测试，我们先来创建一个测试文件。我们在 `test` 目录中新建 `hello_spec.js` 文件，然后往里面添加以下内容：

_test/hello\_spec.js_

```js
describe('Hello', function() {

  it('says hello', function() {
    expect(sayHello()).toBe('Hello, world!');
  });

});
```

如果你曾经用过 Ruby 的 [rSpec](http://rspec.info/) ，应该不会对这种语法感到陌生：顶层作用域有一个 `describe` 代码块，里面可以存放了多个单元测试。每个单元测试则是通过 `it` 代码块进行定义的，每一个代 `it` 码块都包含一个名称和一个测试函数。

> 这种 Jasmine （或 rSpec）使用的语法来源于 [行为驱动型开发模式](http://en.wikipedia.org/wiki/Behavior-driven_development)。`describe` 和 `it` 代码块一同定义了代码的行为。这也是我们为测试文件加上 `_spec.js` 后缀的原因，这些测试文件相当于代码的_说明书_。

下面我们启动 Karma 来执行这个单元测试：

```bash
npm test
```

这行命令会启动 Karma，然后在一个隐形的 PhantomJS 浏览器中进行测试。在 Karma 处于运行状态时，我们可以尝试更改 `src/hello.js` 和 `test/hello_spec.js` 的内容，看 Karma 是否能检测到文件发生了变化并重新运行测试套件。

在你阅读本书的时候，我推荐在终端窗口中一直运行着 Karma。这样你就不用一直通过手动的方式执行这些测试，有代码出错的话我们也能及时发现。

这个单元测试通过了，我们可以继续下一步了。

