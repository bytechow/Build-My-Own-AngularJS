## 使用 Jasmine，Sinon 和 Karma 构建单元测试（Enable Unit Testing With Jasmine, Sinon, and Karma）

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