### 构建产品包（Building The Production Bundle）

现在我们已经拥有了一个可以进行运行测试的框架了！在我们正式进行测试之前，我们先要把框架打包成一个可以在网页中加载的文件。

我们已经在项目中集成了 Browserify，但目前只是把它用于单元测试。而我们现在缺的是一种能构建产品包的方式。

要解决这个问题也很简单。我们只需要把 bootstrap 文件作为参数，运行 Browserify 就可以了：

```bash
./node_modules/browserify/bin/cmd.js src/bootstrap.js
```

注意，当你运行这行命令时，它会把整个包的内容都输出到 STDOUT，因此命令行会出现很多输出内容！

我们可以这样做是因为 Browserify 会读取启动文件，然后从中获取当前文件所需的依赖，因此所有依赖文件都会打包到最后的产品包中。因为`bootstrap.js`依赖`angular_public.js`，而`angular_public.js`依赖我们开发的所有 Angular 框架特性，因此我们会得到一个完整的包。这个包也会包含 LoDash 和 jQuery，因为我们的应用代码使用到了它们。所以这真是一个“包含了电池”的代码库。

如果我们把 Browserify 的输出重定向到一个文件上，我们就会得到一个可以放到网页上的文件：

```bash
./node_modules/browserify/bin/cmd.js src/bootstrap.js > myangular.js
```

如果我们使用 NPM 来运行这条命令会方便很多，所以我们在`package.json`加入一个名为“build” 的运行脚本：

_package.json_

```json
"scripts": {
  "lint": "jshint src test",
  "test": "karma start",
  "build": "browserify src/bootstrap.js > myangular.js"
}
```

现在我们就可以使用这条脚本来生成产品包：

```bash
npm run build
```

这个包是非常大的，因为除了我们写的代码，还加入了 jQuery 和 LoDash 的代码，这样加起来后代码数量就非常惊人了。如果我们可以生成一个经过压缩的、体积更小的版本会更好。

我们可以添加`UglifyJS`压缩工具来解决这个问题：

```bash
npm install --save-dev uglifyjs
```

我们可以引入另一个运行脚本来运行 Browserify，然后把运行结果通过 UglifyJS 来对它进行压缩。最终输出的内容会重定向到一个名为`myangular.min.js`的文件：

_package.json_

```json
"scripts": {
  "lint": "jshint src test",
  "test": "karma start",
  "build": "browserify src/bootstrap.js > myangular.js",
  "build:minified": "browserify src/bootstrap.js | uglifyjs -mc > myangular.min.js"
}
```

注意，当你现在运行`npm run build:minified`时，你会看到有一些警告，这些警告就是说我们的代码里有些东西是 UglifyJS 不希望看到。这是正常的现象，并不会影响我们最终产出文件的实际效用。
