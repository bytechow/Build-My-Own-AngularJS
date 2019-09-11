### 运行一个示例应用（Running An Example App）

现在我们已经可以生成一个产品包了，我们会通过写一个小小的测试应用来验证一下框架是否能正常运作。这是非常激动人心的时刻，因为我们会在测试应用里验证本书开发的所有东西，并能具体展示出我们也能用自己开发的框架做到 AngularJS 能做到的事情！

你可以在电脑上的任何一个地方创建这个示例应用——你要做的只是把`myangular.js`或`myangular.min.js`拷贝过去。最简单的做法是把示例应用放到主项目的子目录中，这样当你需要更改或重新构建应用时，只需要直接指向上层目录的打包文件即可，不需要再拷贝多一次。

我们先创建一个`index.html`文件。它会加载两个脚本：`myangular.js`框架和一个应用文件`app.js`：

_example-app/index.html_

```xml
<!DOCTYPE html>
<html>
<head>
</head>
<body>
  <script src="../myangular.js"></script>
  <script src="app.js"></script>
</body>
</html>
````

同时在当前目录下创建一个`app.js`文件。

如果你现在在浏览器中打开这个页面，页面会正常加载，不会报出错误。我们不需要特地创建一个 HTTP server，在文件系统中打开就可以了。

然后，我们会在这个网页文件中加入一个用于自动启动的属性：

```xml
<body ng-app="myExampleApp">
  <script src="../myangular.js"></script>
  <script src="app.js"></script>
</body>
```

重新加载这个页面时，我们会发现报出一个缺失模块的错误，这就是我们想要有的效果！

如果我们在`app.js`加入这个模块，错误就会消失了：

_example-app/app.js_

```js
angular.module('myExampleApp', []);
```

下一步，我们会试着使用`ngController`指令来绑定一个控制器：

_example-app/index.html_

```xml
<body ng-app="myExampleApp">
  <div ng-controller="ExampleController as ctrl">
  </div>
  <script src="../myangular.js"></script>
  <script src="app.js"></script>
</body>
```

这样做又会报出一个错误，这次错误提示并没有给到我们比较有帮助的信息。与 AngularJS 开发团队相比，我们花费在错误处理上的时间并没有那么多。

但无论如何，只要我们在应用文件中加入了这个控制器，报错就会消失了：

_example-app/app.js_

```js
angular.module('myExampleApp', [])
  .controller('ExampleController', function() {

  });
```

我们再来试试在 DOM 中使用表达式：

_example-app/index.html_

```xml
<div ng-controller="ExampleController as ctrl">
  {{ctrl.counter}}
</div>
```

正如我们希望看到的那样，这个表达式会在应用加载后消失不见，因为现在表达式并没有生成任何值。

如果我们在控制器中为这个表达式加上一个值，这个值就会出现在网页上了：

_example-app/app.js_

```js
angular.module('myExampleApp', [])
  .controller('ExampleController', function() {
    this.counter = 1;
});
```

我们可以在页面上加上几个按钮来增加一些交互元素：一个按钮用于让计数器计数加一，另一个用于减一。这里我们可以使用在本章开头实现的`ngClick`指令：

_example-app/index.html_

```xml
<div ng-controller="ExampleController as ctrl">
  {{ctrl.counter}}
  <button ng-click="ctrl.increment()">+</button>
  <button ng-click="ctrl.decrement()">-</button>
</div>
```

我们在控制器中加入`increment`和`decrement`两个方法，就可以看到计数器变量在发生变化：

_example-app/app.js_

```js
angular.module('myExampleApp', [])
  .controller('ExampleController', function() {
    // this.counter = 1;
    this.increment = function() {
      this.counter++;
    };
    this.decrement = function() {
      this.counter--;
    };
});
```

这样我们就开发了一个简单但拥有完整功能的应用，这个应用是依靠我们从零开发出来的 AngularJS 运行起来的！每一次的点击，从 DOM 处理，到表达式解析，再到 digest 的所有功能都是我们自己写出来的。感觉我们变了一个魔术，不是吗！

你可以继续测试我们开发的其他功能，比如下面这些：

- 创建一些 factory 或 service，然后把它们注入到控制器中
- 启用严格的注入模式
- 在表达式中使用过滤器
- 尝试使用压缩后的包
- 创建一些自定义的指令
- 使用`templateUrl`来加载一些外部模版（注意尝试这个功能你需要启动一个 HTTP 服务器，因为`file://`开头的 URL 不能加载模板）

如果在测试过程中卡住了，你可以使用浏览器的调试器对代码打断点进行调试。这些代码应该都是我们很熟悉的，毕竟是我们把它们写出来的——除了 jQuery 和 LoDash。