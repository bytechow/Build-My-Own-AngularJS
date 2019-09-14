## 利用 JSHint 做静态分析（Enable Static Analysis With JSHint）

JSHint 是一个能够读取 JavaScript 代码然后提供关于代码中出现的语法或结构性错误的分析报告的工具。分析过程被称为_linting_，它对我们十分重要，因为作为一个库的作者，我们并不希望分享一些可能引起问题的代码给他人。

我们可以通过 NPM 获取到 `jshint` 包，那就使用`npm`命令安装这个包下来就可以了：

```bash
npm install --save-dev jshint
```

当你运行这行命令时，会发生几件事：首先一个名为`node_modules`的目录会自动生成在项目内部，而`jshint`包也会下载到这个目录下。

另外，如果你查看一下`package.json`文件，你会看到文件已经发生了改变：现在文件中增加了一个`devDependencies`的 key，而这个 key 对应的对象里面会有一个 `jshint` 的 key。这说明 `jshint` 是我们项目中加入的一个用于开发阶段看的依赖包。造成`package.json`发生如此改变的关键就在于命令中的 `--save-dev` 标识。

现在我们可以为 JSHint 创建一个配置文件了。当我们运行这个工具的时候，它会自动在当前目录中查找一个叫`.jshintrc`的文件，并从文件里面读取配置参数。下面我们创建这个文件，并加入以下内容：

_.jshintrc_

```js
{
  "browser": true,
  "browserify": true,
  "devel": true
}
```

这里我们启用允许 `browser`，`browserify` 和 `devel` 语法的JSHint 环境。这样能确保当我们访问浏览器中的一些常见的全局变量时 JSHint 不会报错，比如`setTimeout`和`console`，或者 Browserify 模块系统中会用到的 `module` 和 `require` 。

现在我们已经做好准备工作了，可以运行 JSHint 了：

```bash
./node_modules/jshint/bin/jshint src
```

看上去我们这个简单的“Hello, world”程序并没有出现静态分析错误！要确保我们的工具是正常运作的，你可以故意让`hello.js`中的代码出错（例如，把 `function` 改成 `funktion`），看看 JSHint 会输出些什么。

我们可以使用 NPM 的_运行脚本_来简化运行 JSHint 的命令。在 `package.json` 中保存了这个命令行之后，以后我们就可以使用这个简化的命令，而不用再记忆那一长串命令了：

_package.json_

```json
{
  "name": "my-own-angularjs",
  "version": "0.1.0",
  "devDependencies": {
    "jshint": "^2.8.0"
  },
  "scripts": {
    "lint": "jshint src"
  }
}
```

从现在起，如果你要对项目里的代码使用 JSHint，你可以直接运行下面的命令：

```bash
npm run lint
```
