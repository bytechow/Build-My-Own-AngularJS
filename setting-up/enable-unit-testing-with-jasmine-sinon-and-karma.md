### 使用 Jasmine，Sinon 和 Karma 构建单元测试（Enable Unit Testing With Jasmine, Sinon, and Karma）

单元测试对于我们整个的开发过程来说是绝对的中心地位，这意味着我们需要一个好的测试框架。我们、使用的测试框架是 [Jasmine](https://jasmine.github.io/2.3/introduction.html)，因为它有良好、简洁的 API，同时也满足我们的需求：

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

实际要运行这些测试时，我们还会用到一个很受欢迎的测试执行器—— [Karma](http://karma-runner.github.io/)。在之后我们安装了它的相关插件之后，Karma 就能与 Jasmine 和 Browserify 结合起来使用。

我们还会用到一个测试帮助库—— [Sinon.JS](https://sinonjs.org/)，这个库能为我们提供一些之后会用到的较为复杂的 mock 对象。具体来说，在我们开始构建 HTTP 服务的功能时，Sinon 会非常有用。

我们先来安装 Jasmine 和 Sinon：

```bash
npm install --save-dev jasmine-core sinon
```

然后，我们再安装 Karma 和一些我们需要用到的 Karma 插件：

```bash
npm install --save-dev karma karma-jasmine karma-jshint-preprocessor
```

还要安装一个 Phantom 影子浏览器，Karma 会在这个隐形的浏览器中运行单元测试：

```bash
npm install --save-dev phantomjs-prebuilt karma-phantomjs-launcher
```

下一步，我们会在 Karme 指定的配置文件 `karma.conf.js` 加载并配置 Karme 和 Jasmine：

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

这样我们就会对 `src` 目录下的所有 JavaScript 文件运行测试。测试本身则存放在 `test` 目录下。当测试运行时，Karma 会时刻监控 `src` 目录下的文件是否发生了改变，如果发生了变化则自动重新运行对应的测试套件。

这里，我们还会设置一个 `jshint` 预处理器，在每个测试套件启动之前，我们都会先运行 JSHint。

单元测试中会用到很多由 Jasmine 定义的全局变量。我们会把全局变量都放到 JSHint 的 `globals` 对象中，以免 JSHint 报错：

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

我们也可以通过添加一个 NPM 脚本来简化 Karma 的启动。在此之前，我们先加入`lint`脚本，它会对 `test` 目录下的进行 jshint 检查：

_package.json_

```json
"scripts": {
  "lint": "jshint src test",
  "test": "karma start"
}
```

现在已经做好了运行单元测试的准备了，让我们创建一个测试文件试一下。我们在 `test` 目录中新建一个叫 `hello_spec.js` 的文件，然后往里面添加以下内容：

_test/hello_spec.js_

```js
describe('Hello', function() {

  it('says hello', function() {
    expect(sayHello()).toBe('Hello, world!');
  });

});
```

如果你曾经用过 Ruby 的 [rSpec](http://rspec.info/) ，应该不会对这种代码格式感到陌生：顶层作用域有一个 `describe` 代码块，里面可以存放了多个单元测试。而单元测试本身是通过 `it` 代码块进行定义的，每一个代码块都包含一个名称和一个测试函数。

> 这种 Jasmine （或 rSpec）使用的语法来源于 [行为驱动型开发模式](http://en.wikipedia.org/wiki/Behavior-driven_development)。`describe` 和 `it` 代码块一同定义了代码的行为。这也是为什么我们为测试文件加上 `_spec.js` 后缀的原因，这些测试文件是对代码的_说明书_。

下面我们启动 Karma 来运行这个单元测试：

```bash
npm test
```

这行命令会启动 Karma，然后在一个隐形的 PhantomJS 浏览器中运行。在 Karma 运行时，我们可以试试更改 `src/hello.js` 和 `test/hello_spec.js`，看看它是否能检测到文件发生了变化，并重新运行测试套件。

在你阅读本书的时候，我推荐你在终端窗口中一直运行着 Karme。这样你不用再手动执行这些测试，而如果有代码出错了我们也能及时发现。

现在我们写的这个单元测试通过了，我们可以继续下一步了。
