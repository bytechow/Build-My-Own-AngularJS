### 安装 Node 和 NPM（Install Node and NPM）

我们的项目将会非常依赖 [Node.js](http://nodejs.org)。这个 JavaScript 运行时将是支撑我们项目构建和测试的基础。Node.js 本身也已经集成了 NPM，而 NPM 则 是我们将要用到的包管理器。

幸运的是，Node 和 NPM 支持各种操作系统，如 Linux，Mac OS X 和 Windows。这里我就不详细介绍 Node 的安装过程了，大家可以到 [https://nodejs.org/download/](https://nodejs.org/download/) 查看官方的安装指南。

在开始创建项目之前，请确保你已经安装了 `node`，并且在终端上能够使用 `npm` 命令。我的电脑在成功安装了 Node 后，终端输出会像下面这样：

```bash
node -v
v5.9.1

npm -v 
3.7.3
```

你电脑的终端输出可能会跟我的不太一样，安装的版本不同，就会有不同的输出结果。但安装了哪个版本并不重要，重要的是有没有出现类似这样的命令行，只要出现了就证明安装成功了。