### 利用 JSHint 做静态分析（Enable Static Analysis With JSHint）

JSHint 是一个能够读取 JavaScript 代码，然后提供关于代码中出现的语法或结构性错误的分析报告的工具。它的分析过程被称为_linting_，这个分析功能对我们十分重要，毕竟作为代码库的作者，我们并不希望产出一些可能会带给别人麻烦的代码。

我们可以通过 NPM 获取到 `jshint` 包，因此直接使用`npm`命令安装这个包下来就可以了：

```bash
npm install --save-dev jshint
```

当你运行这行命令时，会发生几件事：首先一个名为`node_modules`的目录会在项目中自动生成，而`jshint`包会被下载到这个目录下。

另外，如果此刻查看`package.json`文件，你会看到文件已经发生了改变：现在文件中增加了一个`devDependencies`的 key，而这个 key 对应的对象里面会有一个 `jshint` 的 key 存在。意思是 `jshint` 是我们项目中加入的一个仅会开发阶段使用的依赖包。让`package.json`发生这种改变的关键在于上面命令中的 `--save-dev` 参数。

现在我们可以为 JSHint 创建一个配置文件了。当我们运行 JSHint 的时候，JSHint 会自动在当前目录中查找一个叫`.jshintrc`的文件，并从文件里面读取配置参数。创建这个文件后，往里面填充以下内容：

_.jshintrc_

```js
{
  "browser": true,
  "browserify": true,
  "devel": true
}
```

这里我们设置了在 JSHint 环境中允许使用 `browser`，`browserify` 和 `devel` 语法。这样能确保当我们访问浏览器中的一些通用的全局变量时 JSHint 不会报错，类似的全局变量有`setTimeout`和`console`，或者 Browserify 模块系统中会用到的 `module` 和 `require` 。

现在可以运行 JSHint 了：

```bash
./node_modules/jshint/bin/jshint src
```

看上去我们这个简单的“Hello, world”程序经过静态分析后并没有发现错误！但为了确保这个工具是正常运作的，你可以故意让`hello.js`中的代码出错（例如，把 `function` 改成 `funktion`），看看 JSHint 会输出些什么。

我们可以使用 NPM 的 _运行脚本（run script_ 来简化运行 JSHint 的命令。在 `package.json` 中保存这个命令后，以后我们就可以直接使用简写命令，而不用再记忆之前那一长串命令了：

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

从现在起，如果你要对项目里的代码使用 JSHint，可以直接运行下面的命令：

```bash
npm run lint
```



